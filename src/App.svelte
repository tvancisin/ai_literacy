<script>
  import { onMount } from "svelte";
  import {
    forceCollide,
    forceSimulation,
    forceX,
    forceY,
    scalePoint,
  } from "d3";
  import { scale } from "svelte/transition";

  // url handling
  const baseUrl = import.meta.env.BASE_URL;
  function assetUrl(path) {
    return `${baseUrl}${path.replace(/^\//, "")}`;
  }

  // data
  let isLoading = true;
  let loadError = null;
  let progress = { sessions: [], participants: [] };
  $: sessions = progress.sessions ?? [];
  $: participants = progress.participants ?? [];

  // layout
  let height = 400;
  let width = 400;
  $: margin = {
    top: 80,
    right: 50,
    bottom: 40,
    left: 50,
  };

  $: sessionScale = scalePoint()
    .domain(sessions.map((session) => session.id))
    .range([margin.left * 2, width - margin.right])
    .padding(0.35);
  $: participantScale = scalePoint()
    .domain(participants.map((participant) => participant.id))
    .range([margin.top, height - margin.bottom])
    .padding(0.35);
  $: laneStart = margin.left;
  $: laneEnd = width - margin.right;

  // grouped circle packing
  const minMarkerRadius = 10;
  const maxMarkerRadius = 35;
  const collisionPadding = 2;

  function stableRadius(id) {
    let hash = 0;

    for (const character of id) {
      hash = (hash * 31 + character.charCodeAt(0)) % 100000;
    }

    return minMarkerRadius + (hash % (maxMarkerRadius - minMarkerRadius + 1));
  }

  $: rawArtifacts = participants.flatMap((participant) =>
    sessions.flatMap((session) => {
      const sessionArtifacts = participant.sessions?.[session.id] ?? [];

      return sessionArtifacts.map((artifact, index) => ({
        id: artifact.id,
        groupId: session.id,
        participant,
        session,
        artifact,
        artifactIndex: index,
        radius: stableRadius(artifact.id),
      }));
    }),
  );

  function layoutSessionColumns(nodes) {
    const top = margin.top + maxMarkerRadius;
    const bottom = height - margin.bottom - maxMarkerRadius;
    const centerY = (top + bottom) / 2;
    const positioned = nodes
      .map((node, index) => {
        const targetX = sessionScale(node.session.id);
        const angle = ((index * 137.5) % 360) * (Math.PI / 180);
        const radius = 24 + (index % 24) * 3;

        return {
          ...node,
          targetX,
          x: targetX + Math.cos(angle) * radius,
          y: centerY + Math.sin(angle) * radius,
        };
      })
      .filter((node) => node.targetX != null);

    forceSimulation(positioned)
      .alpha(1)
      .alphaDecay(0.018)
      .velocityDecay(0.42)
      .force("x", forceX((node) => node.targetX).strength(0.2))
      .force("y", forceY(centerY).strength(0.035))
      .force(
        "collide",
        forceCollide((node) => node.radius + collisionPadding)
          .strength(1)
          .iterations(8),
      )
      .stop()
      .tick(360);

    return positioned.map((node) => ({
      ...node,
      x: Math.max(node.radius, Math.min(width - node.radius, node.x)),
      y: Math.max(top, Math.min(bottom, node.y)),
    }));
  }

  // positioning of the circles
  $: artifacts =
    rawArtifacts.length && width > 0 && height > 0
      ? layoutSessionColumns(rawArtifacts)
      : [];

  $: sessionColumns = sessions
    .map((session) => {
      const x = sessionScale(session.id);
      const count = artifacts.filter(
        (artifact) => artifact.session.id === session.id,
      ).length;

      return {
        session,
        x,
        count,
      };
    })
    .filter((column) => column.x != null);

  // artifact panel
  let activeArtifactId = null;
  let isPanelExpanded = false;
  $: activeArtifact =
    artifacts.find((artifact) => artifact.id === activeArtifactId) ?? null;
  $: panelWidth = width < 780 ? 244 : 312;
  $: expandedPanelWidth = Math.max(320, width * 0.5);
  $: expandedPanelHeight = Math.max(260, height * 0.5);
  $: lastSessionId = sessions[sessions.length - 1]?.id;
  $: isLastSessionArtifact = activeArtifact?.session.id === lastSessionId;
  $: panelLeft = activeArtifact
    ? isPanelExpanded
      ? (width - expandedPanelWidth) / 2
      : activeArtifact.x > width / 2
        ? Math.max(16, activeArtifact.x - panelWidth - 18)
        : Math.min(activeArtifact.x + 18, width - panelWidth - 16)
    : 0;
  $: panelTop = activeArtifact
    ? isPanelExpanded
      ? (height - expandedPanelHeight) / 2
      : Math.max(14, Math.min(activeArtifact.y - 82, height - 252))
    : 0;
  $: panelStyle = isPanelExpanded
    ? `left: ${panelLeft}px; top: ${panelTop}px; width: ${expandedPanelWidth}px; height: ${expandedPanelHeight}px;`
    : `left: ${panelLeft}px; top: ${panelTop}px; width: ${panelWidth}px;`;

  // interactions 
  function showArtifact(id) {
    activeArtifactId = id;
    isPanelExpanded = false;
  }

  function hideArtifact() {
    activeArtifactId = null;
    isPanelExpanded = false;
  }

  function togglePanelExpanded() {
    isPanelExpanded = !isPanelExpanded;
  }

  function handleDocumentClick(event) {
    if (
      event.target instanceof Element &&
      event.target.closest(".artifact-marker, .artifact-panel")
    ) {
      return;
    }

    hideArtifact();
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

  onMount(() => {
    document.addEventListener("click", handleDocumentClick);

    return () => {
      document.removeEventListener("click", handleDocumentClick);
    };
  });

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

    <!-- session columns -->
    <g class="session-guides" aria-hidden="true">
      {#each sessionColumns as column}
        <line
          x1={column.x}
          x2={column.x}
          y1={margin.top - 30}
          y2={height - margin.bottom}
        />
        <text x={column.x} y={38} text-anchor="middle">
          {column.session.label}
        </text>
        <text
          class="session-topic"
          x={column.x}
          y={58}
          text-anchor="middle"
        >
          {column.session.topic}
        </text>
        <text
          class="session-count"
          x={column.x}
          y={height - 14}
          text-anchor="middle"
        >
          {column.count} artifacts
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
          <!-- <path class="lane-path" d={`M ${laneStart} ${y} H ${laneEnd}`} /> -->
        </g>
      {/each}
    </g>

    <!-- individual artifacts/circles -->
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
          onclick={(event) => {
            event.stopPropagation();
            showArtifact(artifact.id);
          }}
          onkeydown={(event) => handleMarkerKeydown(event, artifact.id)}
        >
          <title>{artifact.participant.name}, {artifact.session.label}</title>
          <circle
            class="marker-ring"
            r={artifact.radius}
            style={`--active-radius: ${artifact.radius + 4}px;`}
          />
          <!-- <circle class="marker-dot" r="4.5" /> -->
        </g>
      {/each}
    </g>
  </svg>

  {#if activeArtifact}
    <aside
      class="artifact-panel"
      class:expanded={isPanelExpanded}
      style={panelStyle}
      transition:scale={{ duration: 160, start: 0.94, opacity: 0 }}
    >
      <button
        class="panel-expand"
        type="button"
        aria-label={isPanelExpanded
          ? "Shrink artifact details"
          : "Expand artifact details"}
        aria-pressed={isPanelExpanded}
        onclick={togglePanelExpanded}
      >
        {isPanelExpanded ? "Shrink" : "Expand"}
      </button>
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
    overflow: hidden;
    overscroll-behavior: none;
    touch-action: pan-y;
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

  .session-guides .session-count {
    fill: #8a9398;
    font-size: 11px;
    font-weight: 600;
  }

  .participant-icon {
    opacity: 0.82;
  }

  .artifact-marker {
    color: #000000;
    cursor: pointer;
    outline: none;
    touch-action: manipulation;
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
    fill: #dfdfdf;
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
    r: var(--active-radius);
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
    padding: 44px 12px 12px;
    border: 1px solid rgba(38, 50, 56, 0.16);
    border-radius: 8px;
    background: rgba(255, 255, 255, 0.97);
    box-shadow: 0 16px 44px rgba(38, 50, 56, 0.16);
    color: #263238;
    transform-origin: center;
    transition:
      left 220ms ease,
      top 220ms ease,
      width 220ms ease,
      height 220ms ease,
      padding 220ms ease,
      gap 220ms ease,
      box-shadow 220ms ease;
  }

  .artifact-panel.expanded {
    grid-template-columns: minmax(180px, 42%) 1fr;
    gap: 20px;
    padding: 56px 24px 24px;
  }

  .panel-expand {
    position: absolute;
    top: 10px;
    right: 10px;
    min-width: 78px;
    height: 30px;
    border: 1px solid rgba(38, 50, 56, 0.18);
    border-radius: 6px;
    background: #ffffff;
    color: #263238;
    cursor: pointer;
    font: inherit;
    font-size: 12px;
    font-weight: 700;
    touch-action: manipulation;
    transition:
      background-color 160ms ease,
      border-color 160ms ease;
  }

  .artifact-panel img {
    width: 96px;
    aspect-ratio: 16 / 9;
    border-radius: 6px;
    object-fit: cover;
    background: #eef2f1;
    transition:
      width 220ms ease,
      height 220ms ease;
  }

  .artifact-panel.expanded img {
    width: 100%;
    height: 100%;
  }

  .artifact-panel h2,
  .artifact-panel p {
    margin: 0;
  }

  .artifact-panel h2 {
    margin-bottom: 5px;
    font-size: 15px;
    line-height: 1.2;
    transition: font-size 220ms ease;
  }

  .artifact-panel p {
    font-size: 12px;
    line-height: 1.45;
    transition: font-size 220ms ease;
  }

  .artifact-panel.expanded h2 {
    font-size: 24px;
  }

  .artifact-panel.expanded p {
    font-size: 18px;
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
