<script>
  import { onMount } from "svelte";
  import { forceCollide, forceSimulation, forceX, forceY, scalePoint } from "d3";

  let isLoading = true;
  let loadError = null;

  let progress = { sessions: [], participants: [] };
  $: sessions = progress.sessions ?? [];
  $: participants = progress.participants ?? [];
  const baseUrl = import.meta.env.BASE_URL;

  let height = 400;
  let width = 400;
  $: margin = {
    top: 80,
    right: 20,
    bottom: 20,
    left: 150,
  };

  $: laneStart = margin.left;
  $: laneEnd = width - margin.right;
  $: sessionScale = scalePoint()
    .domain(sessions.map((session) => session.id))
    .range([laneStart, laneEnd])
    .padding(0.25);
  $: participantScale = scalePoint()
    .domain(participants.map((participant) => participant.id))
    .range([margin.top, height - margin.bottom])
    .padding(0.35);

  const markerRadius = 6;
  const collisionRadius = 8;

  function initialOffset(index, count) {
    if (count === 1) {
      return { x: 0, y: 0 };
    }

    const angle = (index * 137.5 * Math.PI) / 180;
    const distance = 5 + index * 2;

    return {
      x: Math.cos(angle) * distance,
      y: Math.sin(angle) * distance,
    };
  }

  function layoutLocalBeeswarm(sessionArtifacts) {
    const nodes = sessionArtifacts.map((artifact, index) => {
      const offset = initialOffset(index, sessionArtifacts.length);

      return {
        artifact,
        x: offset.x,
        y: offset.y,
      };
    });

    forceSimulation(nodes)
      .force("x", forceX(0).strength(0.1))
      .force("y", forceY(0).strength(0.85))
      .force("collide", forceCollide(collisionRadius))
      .stop()
      .tick(48);

    return nodes;
  }

  $: artifacts = participants.flatMap((participant) =>
    sessions.flatMap((session) => {
      const sessionArtifacts = participant.sessions?.[session.id] ?? [];
      const baseX = sessionScale(session.id);
      const baseY = participantScale(participant.id);
      const groupId = `${participant.id}-${session.id}`;

      if (!sessionArtifacts.length || baseX == null || baseY == null) {
        return [];
      }

      return layoutLocalBeeswarm(sessionArtifacts).map((node) => ({
          id: node.artifact.id,
          groupId,
          participant,
          session,
          artifact: node.artifact,
          x: baseX + node.x,
          y: baseY + node.y,
        }));
    }),
  );

  let activeArtifactId = null;
  $: activeArtifact =
    artifacts.find((artifact) => artifact.id === activeArtifactId) ?? null;
  $: panelWidth = width < 780 ? 244 : 312;
  $: lastSessionId = sessions[sessions.length - 1]?.id;
  $: isLastSessionArtifact = activeArtifact?.session.id === lastSessionId;
  $: panelLeft = activeArtifact
    ? isLastSessionArtifact
      ? Math.max(16, activeArtifact.x - panelWidth - 18)
      : Math.min(activeArtifact.x + 18, width - panelWidth - 16)
    : 0;
  $: panelTop = activeArtifact
    ? Math.max(14, Math.min(activeArtifact.y - 82, height - 252))
    : 0;

  function showArtifact(id) {
    activeArtifactId = id;
  }

  function hideArtifact() {
    activeArtifactId = null;
  }

  function handleMarkerKeydown(event, id) {
    if (event.key === "Enter" || event.key === " ") {
      event.preventDefault();
      showArtifact(id);
    }

    if (event.key === "Escape") {
      hideArtifact();
    }
  }

  function assetUrl(path) {
    return `${baseUrl}${path.replace(/^\//, "")}`;
  }

  onMount(async () => {
    try {
      const response = await fetch(assetUrl("data/progress.json"));

      if (!response.ok) {
        throw new Error(`Request failed with status ${response.status}`);
      }

      progress = await response.json();
    } catch (error) {
      loadError =
        error instanceof Error ? error.message : "Unknown loading error";
    } finally {
      isLoading = false;
    }
  });
</script>

<main bind:clientWidth={width} bind:clientHeight={height}>
  <svg {width} {height}>
    <rect class="chart-background" {width} {height} />

    <!-- sessions vertical indicators -->
    <g class="session-guides" aria-hidden="true">
      {#each sessions as session}
        <line
          x1={sessionScale(session.id)}
          x2={sessionScale(session.id)}
          y1={margin.top - 30}
          y2={height - margin.bottom}
        />
        <text x={sessionScale(session.id)} y={38} text-anchor="middle">
          {session.label}
        </text>
        <text
          class="session-topic"
          x={sessionScale(session.id)}
          y={58}
          text-anchor="middle"
        >
          {session.topic}
        </text>
      {/each}
    </g>

    <!-- horizontal participant lines -->
    <g class="participant-lanes">
      {#each participants as participant}
        {@const y = participantScale(participant.id)}
        <g>
          <image
            class="participant-icon"
            href={assetUrl("img/person.svg")}
            x={margin.left - 42}
            y={y - 12}
            width="24"
            height="24"
            aria-label={participant.name}
          />
          <path class="lane-path" d={`M ${laneStart} ${y} H ${laneEnd}`} />
        </g>
      {/each}
    </g>

    <g class="artifact-markers">
      {#each artifacts as artifact (artifact.id)}
        <g
          class:active={activeArtifactId === artifact.id}
          class:sibling={activeArtifact?.groupId === artifact.groupId &&
            activeArtifactId !== artifact.id}
          class="artifact-marker"
          role="button"
          tabindex="0"
          aria-label={`${artifact.participant.name}, ${artifact.session.label}: ${artifact.artifact.text}`}
          transform={`translate(${artifact.x} ${artifact.y})`}
          onmouseenter={() => showArtifact(artifact.id)}
          onmouseleave={hideArtifact}
          onfocus={() => showArtifact(artifact.id)}
          onblur={hideArtifact}
          onkeydown={(event) => handleMarkerKeydown(event, artifact.id)}
        >
          <title>{artifact.participant.name}, {artifact.session.label}</title>
          <circle class="marker-ring" r={markerRadius} />
          <!-- <circle class="marker-dot" r="4.5" /> -->
        </g>
      {/each}
    </g>
  </svg>

  {#if activeArtifact}
    <aside
      class="artifact-panel"
      style={`left: ${panelLeft}px; top: ${panelTop}px; width: ${panelWidth}px;`}
    >
      <img src={assetUrl(activeArtifact.artifact.image)} alt="" />
      <div>
        <p class="panel-kicker">
          {activeArtifact.participant.name} / {activeArtifact.session.label}
        </p>
        <h2>{activeArtifact.session.topic}</h2>
        <p>{activeArtifact.artifact.text}</p>
      </div>
    </aside>
  {/if}
</main>

<style>
  main {
    width: 100%;
    height: 100vh;
    overflow: auto;
    background: linear-gradient(
        180deg,
        rgba(255, 255, 255, 0.92),
        rgba(246, 247, 248, 0.92)
      ),
      #eef2f1;
  }

  svg {
    display: block;
    font-family:
      Inter,
      ui-sans-serif,
      system-ui,
      -apple-system,
      BlinkMacSystemFont,
      "Segoe UI",
      sans-serif;
  }

  .chart-background {
    fill: #f9faf8;
  }

  .session-guides line {
    stroke: #d9ddd8;
    stroke-dasharray: 4 7;
    stroke-width: 1;
  }

  .session-guides text {
    fill: #263238;
    font-size: 14px;
    font-weight: 700;
  }

  .session-guides .session-topic {
    fill: #69757c;
    font-size: 12px;
    font-weight: 500;
  }

  .participant-icon {
    opacity: 0.82;
  }

  .lane-path {
    fill: none;
    stroke: #c7cec8;
    stroke-linecap: round;
    stroke-width: 2;
  }

  .artifact-marker {
    color: #000000;
    cursor: pointer;
    outline: none;
  }

  /* .artifact-marker:nth-child(4n + 2) {
    color: #6b7fb5;
  }

  .artifact-marker:nth-child(4n + 3) {
    color: #c8663d;
  }

  .artifact-marker:nth-child(4n) {
    color: #8b6f2e;
  } */

  .marker-ring {
    fill: #ffffff;
    stroke: currentColor;
    stroke-width: 1;
    transition:
      r 140ms ease,
      stroke-width 140ms ease;
  }

  /* .marker-dot {
    fill: currentColor;
    pointer-events: none;
  } */

  .artifact-marker:hover .marker-ring,
  .artifact-marker:focus .marker-ring,
  .artifact-marker.active .marker-ring {
    r: 8.5;
    stroke-width: 2.75;
  }

  .artifact-marker.sibling .marker-ring {
    fill: #f2f4f1;
    stroke-width: 2.4;
  }

  .artifact-panel {
    position: absolute;
    z-index: 2;
    display: grid;
    grid-template-columns: 96px 1fr;
    gap: 12px;
    padding: 12px;
    border: 1px solid rgba(38, 50, 56, 0.16);
    border-radius: 8px;
    background: rgba(255, 255, 255, 0.97);
    box-shadow: 0 16px 44px rgba(38, 50, 56, 0.16);
    color: #263238;
  }

  .artifact-panel img {
    width: 96px;
    aspect-ratio: 16 / 9;
    border-radius: 6px;
    object-fit: cover;
    background: #eef2f1;
  }

  .artifact-panel h2,
  .artifact-panel p {
    margin: 0;
  }

  .artifact-panel h2 {
    margin-bottom: 5px;
    font-size: 15px;
    line-height: 1.2;
  }

  .artifact-panel p {
    font-size: 12px;
    line-height: 1.45;
  }

  .artifact-panel .panel-kicker {
    margin-bottom: 4px;
    color: #647078;
    font-size: 11px;
    font-weight: 700;
    letter-spacing: 0;
    text-transform: uppercase;
  }

  @media (max-width: 779px) {
    .artifact-panel {
      grid-template-columns: 1fr;
    }

    .artifact-panel img {
      width: 100%;
    }
  }
</style>
