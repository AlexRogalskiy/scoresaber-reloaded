<script>
  import {LEADERBOARD_SCORES_PER_PAGE} from '../../utils/scoresaber/consts'
  import {LEADERBOARD_SCORES_PER_PAGE as ACCSABER_LEADERBOARD_SCORES_PER_PAGE} from '../../utils/accsaber/consts'
  import {opt} from '../../utils/js'
  import BeatSaviorDetails from '../BeatSavior/Details.svelte'
  import LeaderboardPage from '../../pages/Leaderboard.svelte'
  import Switcher from '../Common/Switcher.svelte'

  export let playerId;
  export let songScore;
  export let fixedBrowserTitle = null;
  export let noBeatSaviorHistory = false;
  export let noSsLeaderboard = false;
  export let showAccSaberLeaderboard = false;

  const switcherOptions = [];
  switcherOptions.push({id: 'beatsavior', label: 'Beat Savior', icon: '<div class="beatsavior-icon"></div>'});
  if (showAccSaberLeaderboard) switcherOptions.push({id: 'accsaber', label: 'Leaderboard', icon: '<div class="accsaber-icon"></div>'})
  if (!noSsLeaderboard) switcherOptions.push({id: 'leaderboard', label: 'Leaderboard', iconFa: 'fas fa-cubes'})

  let selectedOption = switcherOptions[0];
  let inBuiltLeaderboardPage = null;

  function getAvailableOptions(songScore) {
    if (!songScore) return null;

    const options = switcherOptions.filter(o => o.id !== 'beatsavior' || songScore.beatSavior);

    if (!options.includes(selectedOption)) {
      selectedOption = options.length ? options[0] : null;
    }

    return options;
  }

  function onOptionChanged(event) {
    selectedOption = event.detail;
  }

  function updateInBuiltLeaderboardPage(rank, type) {
    if (!rank) {
      inBuiltLeaderboardPage = null;
      return;
    }

    inBuiltLeaderboardPage = Math.floor((rank - 1) / (type === 'accsaber' ? ACCSABER_LEADERBOARD_SCORES_PER_PAGE : LEADERBOARD_SCORES_PER_PAGE)) + 1;
  }

  function onInBuiltLeaderboardPageChanged(event) {
    const newPage = opt(event, 'detail.page');
    if (!Number.isFinite(newPage)) return;

    inBuiltLeaderboardPage = newPage;
  }

  $: leaderboard = opt(songScore, 'leaderboard', null);
  $: score = opt(songScore, 'score', null);
  $: prevScore = opt(songScore, 'prevScore', null);
  $: beatSavior = opt(songScore, 'beatSavior', null)

  $: filteredOptions = getAvailableOptions(songScore);
  $: updateInBuiltLeaderboardPage(score && score.rank ? score.rank : null, selectedOption?.id ?? 'leaderboard')
</script>

<section class="details">
  {#if songScore}
    {#if filteredOptions && filteredOptions.length > 1}
      <nav>
        <Switcher values={filteredOptions} value={selectedOption} on:change={onOptionChanged}/>
      </nav>
    {/if}

    <div class="tab">
      {#if selectedOption && selectedOption.id === 'beatsavior'}
        <BeatSaviorDetails {playerId} {beatSavior} {leaderboard} noHistory={noBeatSaviorHistory}/>
      {/if}

      {#if selectedOption && selectedOption.id === 'accsaber'}
        <LeaderboardPage leaderboardId={leaderboard.leaderboardId}
                         type="accsaber"
                         page={inBuiltLeaderboardPage}
                         scrollOffset={176}
                         dontNavigate={true} withoutDiffSwitcher={true} withoutHeader={true}
                         on:page-changed={onInBuiltLeaderboardPageChanged}
                         {fixedBrowserTitle}

        />
      {/if}

      {#if selectedOption && selectedOption.id === 'leaderboard'}
        <LeaderboardPage leaderboardId={leaderboard.leaderboardId}
                         type="global"
                         page={inBuiltLeaderboardPage}
                         scrollOffset={176}
                         dontNavigate={true} withoutDiffSwitcher={true} withoutHeader={true}
                         on:page-changed={onInBuiltLeaderboardPageChanged}
                         {fixedBrowserTitle}

        />
      {/if}
    </div>
  {/if}
</section>


<style>
    .details {
        padding: 1rem 0;
    }

    nav {
        margin-bottom: 1rem;
    }

    .tab {
        display: grid;
        grid-template-columns: 1fr;
        grid-template-rows: 1fr;
        overflow: hidden;
    }

    .tab > :global(*) {
        grid-area: 1 / 1 / 1 / 1;
    }
</style>