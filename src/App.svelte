<script>
  import { onMount } from "svelte";
  import {
    forceCollide,
    forceSimulation,
    forceX,
    forceY,
    scalePoint,
  } from "d3";

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
    top: 150,
    right: 50,
    bottom: 40,
    left: 90,
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
  const minMarkerRadius = 15;
  const maxMarkerRadius = 45;
  const collisionPadding = 2;

  function stableRadius(id) {
    let hash = 0;

    for (const character of id) {
      hash = (hash * 31 + character.charCodeAt(0)) % 100000;
    }

    return minMarkerRadius + (hash % (maxMarkerRadius - minMarkerRadius + 1));
  }

  function markerTextSize(radius) {
    return Math.max(7, Math.min(12, radius * 0.42));
  }

  function activeMarkerRadius(artifact) {
    if (!artifact.hasText) {
      return Math.max(artifact.radius * 2.2, 78);
    }

    const textLength = artifact.text.length;

    return Math.max(
      artifact.radius * 2.2,
      Math.min(150, 72 + textLength * 0.45),
    );
  }

  $: rawArtifacts = participants.flatMap((participant) =>
    sessions.flatMap((session) => {
      const sessionArtifacts = participant.sessions?.[session.id] ?? [];

      return sessionArtifacts.map((artifact, index) => {
        const radius = stableRadius(artifact.id);
        const text = artifact.text?.trim() ?? "";
        const expandedRadius = activeMarkerRadius({ radius, text, hasText: text.length > 0 });

        return {
          id: artifact.id,
          participant,
          session,
          artifact,
          artifactIndex: index,
          radius,
          text,
          hasText: text.length > 0,
          expandedRadius,
          markerTextSize: markerTextSize(radius),
          markerScale: radius / expandedRadius,
        };
      });
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
  $: visibleArtifacts = [...artifacts].sort(
    (a, b) =>
      Number(a.id === activeArtifactId) - Number(b.id === activeArtifactId),
  );

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

  // active artifact
  let activeArtifactId = null;

  // interactions
  function showArtifact(id) {
    activeArtifactId = id;
  }

  function hideArtifact() {
    activeArtifactId = null;
  }

  function handleDocumentClick(event) {
    if (
      event.target instanceof Element &&
      event.target.closest(".artifact-marker")
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
  <img class="efi_logo" src={assetUrl("efi_logo.png")} alt="EFI logo" />
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
        <text x={column.x} y={110} text-anchor="middle">
          {column.session.label}
        </text>
        <!-- <text class="session-topic" x={column.x} y={80} text-anchor="middle">
          {column.session.topic}
        </text> -->
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
          <path class="lane-path" d={`M ${laneStart} ${y} H ${laneEnd}`} />
        </g>
      {/each}
    </g>

    <!-- individual artifacts/circles -->
    <g class="artifact-markers">
      {#each visibleArtifacts as artifact (artifact.id)}
        <g
          class:active={activeArtifactId === artifact.id}
          class="artifact-marker"
          role="button"
          tabindex="0"
          aria-label={`${artifact.participant.name}, ${artifact.session.label}${artifact.hasText ? `: ${artifact.text}` : ""}`}
          transform={`translate(${artifact.x} ${artifact.y})`}
          style={`--marker-scale: ${activeArtifactId === artifact.id ? 1 : artifact.markerScale}; --content-size: ${(artifact.expandedRadius - 1) * 2}px; --content-offset: ${-artifact.expandedRadius + 1}px; --marker-font-size: ${activeArtifactId === artifact.id ? 11 : artifact.markerTextSize / artifact.markerScale}px;`}
          onclick={(event) => {
            event.stopPropagation();
            showArtifact(artifact.id);
          }}
          onkeydown={(event) => handleMarkerKeydown(event, artifact.id)}
        >
          <title>
            {artifact.participant.name}, {artifact.session.label}{artifact.hasText
              ? `: ${artifact.text}`
              : ""}
          </title>
          <g class="marker-visual">
            <circle class="marker-fill" r={artifact.expandedRadius} />
            <foreignObject
              class="marker-content"
              x={-artifact.expandedRadius + 1}
              y={-artifact.expandedRadius + 1}
              width={(artifact.expandedRadius - 1) * 2}
              height={(artifact.expandedRadius - 1) * 2}
            >
              <div
                class:has-text={artifact.hasText}
                class="marker-content-inner"
                style={`background-image: ${artifact.hasText ? "none" : `url(${assetUrl("uni_logo.png")})`};`}
              >
                {#if artifact.hasText}
                  {artifact.text}
                {/if}
              </div>
            </foreignObject>
            <circle class="marker-ring" r={artifact.expandedRadius} />
          </g>
          <!-- <circle class="marker-dot" r="4.5" /> -->
        </g>
      {/each}
    </g>
  </svg>

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

  .efi_logo {
    position: absolute;
    top: 5px;
    left: 5px;
    width: 320px;
    height: auto;
    z-index: 3;
  }

  svg {
    display: block;
    font-family:
      Montserrat,
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
    --marker-scale: 1;
    --marker-font-size: 8px;
    --marker-padding: 0 4px;
    --marker-line-height: 1;
  }

  .marker-visual {
    transform: scale(var(--marker-scale));
    transform-box: view-box;
    transform-origin: 0 0;
    transition: transform 280ms cubic-bezier(0.22, 1, 0.36, 1);
  }

  .marker-fill {
    fill: #fffefe;
    filter: drop-shadow(0 2px 5px rgba(38, 50, 56, 0.18));
    pointer-events: visiblePainted;
  }

  .marker-content {
    pointer-events: none;
  }

  .marker-content-inner {
    display: flex;
    width: 100%;
    height: 100%;
    align-items: center;
    overflow: hidden;
    border-radius: 50%;
    background-position: center;
    background-size: cover;
    color: #263238;
    font-family:
      Montserrat,
      ui-sans-serif,
      system-ui,
      -apple-system,
      BlinkMacSystemFont,
      "Segoe UI",
      sans-serif;
    font-size: var(--marker-font-size);
    font-weight: 700;
    letter-spacing: 0;
    line-height: var(--marker-line-height);
    padding: var(--marker-padding);
    text-shadow:
      0 1px 0 rgba(255, 255, 255, 0.86),
      1px 0 0 rgba(255, 255, 255, 0.86),
      0 -1px 0 rgba(255, 255, 255, 0.86),
      -1px 0 0 rgba(255, 255, 255, 0.86);
    user-select: none;
    white-space: nowrap;
    transition:
      font-size 180ms ease,
      line-height 180ms ease,
      padding 180ms ease;
  }

  .marker-content-inner.has-text {
    padding-left: 4px;
  }

  .artifact-marker.active {
    --marker-padding: 18px;
    --marker-line-height: 1.25;
  }

  .artifact-marker.active .marker-content-inner.has-text {
    justify-content: center;
    text-align: center;
    white-space: normal;
  }

  .marker-ring {
    fill: none;
    stroke: currentColor;
    stroke-width: 0;
    pointer-events: none;
    transition: stroke-width 140ms ease;
  }

  /* .marker-dot {
    fill: currentColor;
    pointer-events: none;
  } */

  .artifact-marker:hover .marker-ring,
  .artifact-marker:focus .marker-ring,
  .artifact-marker.active .marker-ring {
    stroke-width: 2.75;
  }

</style>
