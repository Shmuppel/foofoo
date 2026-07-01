<script lang="ts">
  import { onMount } from 'svelte'

  type Plant = {
    id: string
    name: string
    emoji: string
    lastWatered?: string
    everyDays?: number
    dueStart?: string
    dueEnd?: string
    note?: string
    checkWhenDry?: boolean
  }

  const plants: Plant[] = [
    {
      id: 'monstera',
      name: 'Monstera',
      emoji: '🌿',
      lastWatered: '2026-06-19',
      everyDays: 14,
      note:
        'Take it out of the ceramic pot and put it on the draining plate thingy. Otherwise it will not take up all the water properly and the water will just run through.',
    },
    {
      id: 'tv-plant',
      name: 'Plant on the pedestal next to the TV',
      emoji: '🪴',
      dueStart: '2026-07-04',
      dueEnd: '2026-07-05',
      note: 'Water somewhere during the weekend, 4–5 July, if possible. Put it in the sink and water thoroughly.',
    },
    {
      id: 'vitrine-plant',
      name: 'Plant on top of the vitrine',
      emoji: '🌱',
      dueStart: '2026-07-12',
      note: 'Might leak with the basket so you can put it in the sink if you want. It does not have a good way to catch the water.',
    },
    {
      id: 'other-plants',
      name: 'All other plants',
      emoji: '🌿',
      lastWatered: '2026-06-28',
      checkWhenDry: true,
      note: 'They were watered last Sunday. Water again on Sundays or when the soil gets dry.',
    },
    {
      id: 'hanging-plant',
      name: 'Hanging plant in front of the window',
      emoji: '🌱',
      lastWatered: '2026-06-28',
      checkWhenDry: true,
      note: 'Was watered with the other plants last Sunday. Same idea as the other plants, but maybe hang it above the sink if you want to prevent leaking.',
    },
  ]

  const compliments = [
    'You are doing amazing and the dragon cows are extremely proud of you.',
    'Me, the plants, Jesko, Vilja, Oskar, and the dishwasher believe in you.',
    'You have handled harder quests than this, noble floof.',
    'You are better than a lot of veterinarians, knowing you I fully believe that',
    'Älskar dig jättemycket!'
  ]

  let wateredDates: Record<string, string> = {}
  let today = new Date()

  let showCompliment = false

  let compliment = compliments[0]
  

  onMount(() => {
    const saved = localStorage.getItem('plant-watered-dates')
    wateredDates = saved ? JSON.parse(saved) : {}
    today = new Date()
  })

  function pickRandomCompliment() {
    let nextCompliment = compliments[Math.floor(Math.random() * compliments.length)]

    if (compliments.length > 1) {
      while (nextCompliment === compliment) {
        nextCompliment = compliments[Math.floor(Math.random() * compliments.length)]
      }
    }

    compliment = nextCompliment
    showCompliment = true
  }

  function formatDate(date: Date) {
    return date.toISOString().slice(0, 10)
  }

  function prettyDate(dateString: string) {
    return new Intl.DateTimeFormat('en-GB', {
      day: 'numeric',
      month: 'long',
    }).format(new Date(`${dateString}T12:00:00`))
  }

  function getSavedWateredDate(plant: Plant) {
    return wateredDates[plant.id]
  }

  function getLastWatered(plant: Plant) {
    return getSavedWateredDate(plant) ?? plant.lastWatered
  }

  function getNextWatering(plant: Plant) {
    const savedWateredDate = getSavedWateredDate(plant)

    if (savedWateredDate && plant.everyDays) {
      const date = new Date(`${savedWateredDate}T12:00:00`)
      date.setDate(date.getDate() + plant.everyDays)
      return date
    }

    if (plant.dueStart && !savedWateredDate) {
      return new Date(`${plant.dueStart}T12:00:00`)
    }

    const lastWatered = getLastWatered(plant)

    if (lastWatered && plant.everyDays) {
      const date = new Date(`${lastWatered}T12:00:00`)
      date.setDate(date.getDate() + plant.everyDays)
      return date
    }

    return null
  }

  function getDaysUntil(date: Date) {
    const start = new Date(today)
    start.setHours(0, 0, 0, 0)

    const end = new Date(date)
    end.setHours(0, 0, 0, 0)

    return Math.round((end.getTime() - start.getTime()) / 86_400_000)
  }

  function isToday(dateString: string) {
    return dateString === formatDate(today)
  }

  function isWithinDueWindow(plant: Plant) {
    if (!plant.dueStart || !plant.dueEnd) return false

    const start = new Date(`${plant.dueStart}T00:00:00`)
    const end = new Date(`${plant.dueEnd}T23:59:59`)
    const current = new Date(today)

    return current >= start && current <= end
  }

  function getStatus(plant: Plant) {
    const savedWateredDate = getSavedWateredDate(plant)

    if (savedWateredDate && !plant.everyDays) {
      if (isToday(savedWateredDate)) return 'watered today'
      return `watered on ${prettyDate(savedWateredDate)}`
    }

    if (plant.checkWhenDry) {
      if (savedWateredDate && isToday(savedWateredDate)) return 'watered today'
      return 'check soil and water when dry'
    }

    if (plant.dueStart && plant.dueEnd && isWithinDueWindow(plant)) {
      return 'water this weekend'
    }

    const nextWatering = getNextWatering(plant)

    if (!nextWatering) {
      return 'no date set'
    }

    const daysUntil = getDaysUntil(nextWatering)

    if (daysUntil < 0) {
      const days = Math.abs(daysUntil)
      return `overdue by ${days} day${days === 1 ? '' : 's'}`
    }

    if (daysUntil === 0) return 'water today'
    if (daysUntil === 1) return 'water tomorrow'

    return `in ${daysUntil} days`
  }

  function getSecondaryInfo(plant: Plant) {
    if (plant.dueStart && plant.dueEnd) {
      return `Due ${prettyDate(plant.dueStart)}–${prettyDate(plant.dueEnd)}`
    }

    if (plant.dueStart) {
      return `Due ${prettyDate(plant.dueStart)}`
    }

    const lastWatered = getLastWatered(plant)

    if (lastWatered && plant.everyDays) {
      return `Last watered ${prettyDate(lastWatered)} · every ${plant.everyDays} days`
    }

    if (lastWatered) {
      return `Last watered ${prettyDate(lastWatered)}`
    }

    return ''
  }

  $: sortedPlants = [...plants].sort((a, b) => {
    const nextA = getNextWatering(a)
    const nextB = getNextWatering(b)

    if (!nextA && !nextB) return 0
    if (!nextA) return 1
    if (!nextB) return -1

    return nextA.getTime() - nextB.getTime()
  })
</script>

<main class="page">
  <div class="sparkles" aria-hidden="true"></div>

  <section class="card">
    <p class="eyebrow">They don't make them like you anymore</p>

    <h1>Hej Floof!</h1>

    <p class="intro">
      A little place for floofs that love cows and dragons
    </p>

    <div class="emergency-box">
      <button
        class="emergency-button"
        type="button"
        on:click={pickRandomCompliment}
      >
        Emergency encouragement
      </button>

      {#if showCompliment}
        <p class="compliment">
          {compliment}
        </p>
      {/if}
    </div>

    <section class="video-card" aria-label="Video note">
      <video controls playsinline preload="metadata">
        <source src="/videos/note.mp4" type="video/mp4" />
        <p>Your browser does not support the video tag.</p>
      </video>
    </section>

    <section class="dishwasher-card" aria-label="Dishwasher instructions">
      <div class="dishwasher-text">
        <p class="label">dishwasher guide</p>
        <h2>How to turn it on</h2>
        <p>
          A small video note for the dishwasher, because machines should simply behave.
        </p>
      </div>

      <div class="vertical-video-frame">
        <video controls playsinline preload="metadata">
          <source src="/videos/dishwasher.webm" type="video/webm" />
          <source src="/videos/dishwasher.mp4" type="video/mp4" />
          <p>Your browser does not support the video tag.</p>
        </video>
      </div>
    </section>

    <section class="plants-card" aria-label="Plant watering reminders">
      <div class="plants-header">
        <div>
          <p class="label">plant reminders</p>
          <h2>Watering schedule</h2>
        </div>
      </div>

      <div class="plant-list">
        {#each sortedPlants as plant}
          <article class="plant-item">
            <div class="plant-main">
              <span class="plant-emoji">{plant.emoji}</span>

              <div class="plant-text">
                <div class="plant-title-row">
                  <h3>{plant.name}</h3>
                  <span class="status">{getStatus(plant)}</span>
                </div>

                {#if getSecondaryInfo(plant)}
                  <p class="secondary">{getSecondaryInfo(plant)}</p>
                {/if}

                {#if plant.note}
                  <p class="plant-note">{plant.note}</p>
                {/if}
              </div>
            </div>
          </article>
        {/each}
      </div>
    </section>
  </section>
</main>

<style>
  * {
    box-sizing: border-box;
  }

  :global(html) {
    min-height: 100%;
  }

  :global(body) {
    margin: 0;
    min-height: 100%;
    font-family:
      Inter,
      ui-sans-serif,
      system-ui,
      -apple-system,
      BlinkMacSystemFont,
      'Segoe UI',
      sans-serif;
    color: #2f3428;
    background: #f8efd8;
  }

  button,
  video {
    font: inherit;
  }

  .page {
    position: relative;
    min-height: 100vh;
    display: grid;
    place-items: start center;
    padding: 2rem;

    background-image: url('/background.webp');
    background-size: cover;
    background-position: center;
    background-repeat: no-repeat;
    background-attachment: fixed;
  }

  .page::before {
    content: '';
    position: absolute;
    inset: 0;
    z-index: 0;
    pointer-events: none;
    background:
      radial-gradient(
        circle at center,
        rgb(255 248 225 / 0.38),
        rgb(255 248 225 / 0.12) 48%,
        rgb(255 248 225 / 0.03)
      );
  }

  .sparkles {
    position: fixed;
    inset: 0;
    z-index: 0;
    pointer-events: none;
    background-image:
      radial-gradient(circle, rgb(255 255 255 / 0.9) 0 1px, transparent 2px),
      radial-gradient(circle, rgb(255 238 180 / 0.75) 0 1px, transparent 2px),
      radial-gradient(circle, rgb(255 255 255 / 0.55) 0 1px, transparent 2px);
    background-size:
      90px 90px,
      130px 130px,
      170px 170px;
    animation: sparkle-drift 22s linear infinite;
    opacity: 0.45;
  }

  @keyframes sparkle-drift {
    from {
      background-position:
        0 0,
        0 0,
        0 0;
    }

    to {
      background-position:
        90px 90px,
        -130px 130px,
        170px -170px;
    }
  }

  .card {
    position: relative;
    z-index: 1;
    width: min(100%, 780px);
    padding: clamp(1.5rem, 4vw, 3rem);
    border: 1px solid rgb(255 255 255 / 0.55);
    border-radius: 2rem;
    background: rgb(255 252 240 / 0.72);
    backdrop-filter: blur(18px);
    box-shadow: 0 30px 80px rgb(90 70 40 / 0.16);
  }

  .eyebrow,
  .label {
    margin: 0;
    font-size: 0.75rem;
    font-weight: 800;
    letter-spacing: 0.12em;
    text-transform: uppercase;
    color: #72845f;
  }

  h1,
  h2,
  h3,
  p {
    margin: 0;
  }

  h1 {
    margin-top: 0.4rem;
    font-size: clamp(2.4rem, 8vw, 4.8rem);
    line-height: 0.95;
    letter-spacing: -0.06em;
  }

  .intro {
    max-width: 34rem;
    margin-top: 1rem;
    color: #686c5a;
    font-size: 1.08rem;
    line-height: 1.6;
  }

  .emergency-box {
    margin-top: 1rem;
  }

  .emergency-button {
    border: 0;
    border-radius: 999px;
    padding: 0.7rem 0.95rem;
    background: #f4a259;
    color: #fff8ed;
    font-size: 0.88rem;
    font-weight: 800;
    cursor: pointer;
    box-shadow: 0 12px 28px rgb(130 75 25 / 0.18);
  }

  .emergency-button:hover {
    background: #e88f3f;
  }

  .emergency-button:active {
    transform: translateY(1px);
  }

  .compliment {
    max-width: 32rem;
    margin-top: 0.75rem;
    color: #6d705f;
    font-size: 0.95rem;
    line-height: 1.5;
    font-style: italic;
  }

  .video-card {
    margin-top: 1.5rem;
  }

  .video-card video {
    display: block;
    width: 100%;
    border-radius: 1.35rem;
    background: #111;
    box-shadow: 0 20px 50px rgb(90 70 40 / 0.18);
  }

  .dishwasher-card {
    display: grid;
    grid-template-columns: 1fr auto;
    align-items: center;
    gap: 1.25rem;
    margin-top: 1.5rem;
    padding: 1.25rem;
    border-radius: 1.5rem;
    background: rgb(255 255 255 / 0.72);
    box-shadow: 0 20px 50px rgb(90 70 40 / 0.1);
  }

  .dishwasher-text h2 {
    margin-top: 0.2rem;
    font-size: 1.4rem;
    letter-spacing: -0.03em;
  }

  .dishwasher-text p:not(.label) {
    max-width: 26rem;
    margin-top: 0.45rem;
    color: #6d705f;
    font-size: 0.95rem;
    line-height: 1.45;
  }

  .vertical-video-frame {
    width: min(180px, 32vw);
    aspect-ratio: 9 / 16;
    padding: 0.4rem;
    border: 2px solid transparent;
    border-radius: 1.4rem;
    background:
      linear-gradient(rgb(255 250 235 / 0.96), rgb(255 250 235 / 0.96)) padding-box,
      linear-gradient(135deg, #dceec7, #f5d4a6, #e8b6ad) border-box;
    box-shadow: 0 18px 40px rgb(90 70 40 / 0.16);
  }

  .vertical-video-frame video {
    display: block;
    width: 100%;
    height: 100%;
    border-radius: 1rem;
    object-fit: cover;
    background: #111;
  }

  .plants-card {
    margin-top: 1.5rem;
    padding: 1.25rem;
    border-radius: 1.5rem;
    background: rgb(255 255 255 / 0.72);
    box-shadow: 0 20px 50px rgb(90 70 40 / 0.1);
  }

  .plants-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 1rem;
    margin-bottom: 1rem;
  }

  .plants-header h2 {
    margin-top: 0.2rem;
    font-size: 1.4rem;
    letter-spacing: -0.03em;
  }

  .plant-list {
    display: grid;
    gap: 0.75rem;
  }

  .plant-item {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 1rem;
    padding: 0.9rem;
    border-radius: 1rem;
    background: #fffaf0;
  }

  .plant-main {
    display: flex;
    align-items: flex-start;
    gap: 0.75rem;
    min-width: 0;
  }

  .plant-emoji {
    flex: 0 0 auto;
    font-size: 1.6rem;
    line-height: 1.2;
  }

  .plant-text {
    min-width: 0;
  }

  .plant-title-row {
    display: flex;
    align-items: center;
    flex-wrap: wrap;
    gap: 0.45rem;
  }

  .plant-item h3 {
    font-size: 1rem;
  }

  .status {
    width: max-content;
    border-radius: 999px;
    padding: 0.2rem 0.5rem;
    background: #e5f3d4;
    color: #41542f;
    font-size: 0.75rem;
    font-weight: 800;
  }

  .secondary {
    margin-top: 0.25rem;
    color: #6d705f;
    font-size: 0.9rem;
  }

  .plant-note {
    max-width: 34rem;
    margin-top: 0.4rem;
    color: #7d765f;
    font-size: 0.82rem;
    line-height: 1.45;
  }

  @media (max-width: 640px) {
    .page {
      padding: 1rem;
      background-position: center top;
      background-attachment: scroll;
    }

    .card {
      border-radius: 1.5rem;
    }

    .dishwasher-card {
      grid-template-columns: 1fr;
    }

    .vertical-video-frame {
      width: min(260px, 100%);
      justify-self: center;
    }

    .plant-item {
      flex-direction: column;
    }
  }
</style>