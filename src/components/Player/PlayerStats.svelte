<script>
  import {navigate} from 'svelte-routing'
  import {SS_HOST} from '../../network/queues/scoresaber/page-queue'
  import {PLAYERS_PER_PAGE} from '../../utils/scoresaber/consts'
  import {convertArrayToObjectByKey, opt} from '../../utils/js'

  import Value from '../Common/Value.svelte'
  import Status from './Status.svelte'
  import Skeleton from '../Common/Skeleton.svelte'
  import Error from '../Common/Error.svelte'
  import Badge from '../Common/Badge.svelte'
  import {addToDate, DAY, formatDateRelative} from '../../utils/date'

  export let name;
  export let playerInfo;
  export let prevInfo;
  export let skeleton = false;
  export let centered = false;
  export let error = null;

  function getCountryRankingUrl(countryObj) {
    const rank = opt(countryObj, 'rankValue', opt(countryObj, 'rank', null));
    if (!rank) return null;

    const country = opt(countryObj, 'country', null);
    if (!country) return null;

    return `/ranking/${country.toLowerCase()}/${Math.floor((rank - 1) / PLAYERS_PER_PAGE) + 1}`;
  }

  function navigateToCountryRanking(countryObj) {
    const url = getCountryRankingUrl(countryObj);

    if (url && url.length) navigate(url)
  }

  function navigateToGlobalRanking(rank) {
    if (!rank) return;

    navigate(`/ranking/global/${Math.floor((rank - 1) / PLAYERS_PER_PAGE) + 1}`)
  }

  function getPlayerCountries(playerInfo, prevInfo) {
    if (!playerInfo?.countries) return [];

    const prevCountries = convertArrayToObjectByKey(prevInfo?.countries ?? [], 'country');
    return playerInfo.countries
      .map(c => ({...c, prevRank: prevCountries?.[c.country]?.rank ?? null}));
  }

  $: rank = playerInfo ? (playerInfo.rankValue ? playerInfo.rankValue : playerInfo.rank) : null;
  $: playerRole = playerInfo?.role ?? null;
  $: countries = getPlayerCountries(playerInfo, prevInfo)
  $: gainDate = Number.isFinite(prevInfo?.gainDaysAgo) ? formatDateRelative(addToDate(-prevInfo.gainDaysAgo * DAY)) : null
</script>

{#if skeleton}
  <h1 class="title is=4 has-text-centered-mobile" class:centered>
    <Skeleton width="50%"/>
  </h1>
  <h2 class="title is-5" class:centered>
    <Skeleton width="50%"/>
  </h2>
{:else if playerInfo}
  <h1 class="title is-4 has-text-centered-mobile" class:centered>
    {#if name}
      {#if playerInfo.externalProfileUrl}
        <a href={playerInfo.externalProfileUrl} target="_blank" rel="noreferrer">{name}</a>
      {:else}
        {name}
      {/if}
    {/if}

    <span class="pp">
      <Value value={opt(playerInfo, 'pp')} suffix="pp"
             prevValue={opt(prevInfo, 'pp')} prevLabel={prevInfo?.pp && gainDate ? gainDate : null}
             inline={true} zero="0pp"
      />
    </span>

    <span class="status"><Status {playerInfo}/></span>
  </h1>

  <h2 class="title is-5" class:centered>
    <a href={`/ranking/global/${Math.floor((rank-1) / PLAYERS_PER_PAGE) + 1}`}
       on:click|preventDefault={() => navigateToGlobalRanking(rank)} title="Go to global ranking"
       class="clickable">
      <i class="fas fa-globe-americas"></i>

      <Value value={opt(playerInfo, 'rank')}
             prevValue={opt(prevInfo, 'rank')} prevLabel={prevInfo?.rank && gainDate ? gainDate : null}
             prefix="#" digits={0} zero="#0" inline={true} reversePrevSign={true}
      />
    </a>

    {#each countries as country}
      <a href={getCountryRankingUrl(country)} on:click|preventDefault={() => navigateToCountryRanking(country)}
         title="Go to country ranking" class="clickable">
        <img
          src={`${SS_HOST}/imports/images/flags/${country && country.country && country.country.toLowerCase ? country.country.toLowerCase() : ''}.png`}
             alt={opt(country, 'country')}
        />

        <Value value={country.rank}
               prevValue={country.prevRank} prevLabel={country?.prevRank && gainDate ? gainDate : null}
               prefix="#" digits={0} zero="#0" inline={true}
               reversePrevSign={true}
        />

        {#if country.subRank && country.subRank !== country.rankValue}
          <small>(#{ country.subRank })</small>
        {/if}
      </a>
    {/each}
  </h2>

  {#if playerRole}
    <div class="player-role up-to-tablet">
      <Badge label={playerRole} onlyLabel={true} fluid={true} bgColor="var(--dimmed)" />
    </div>
  {/if}

  {#if error}
    <div>
      <Error {error}/>
    </div>
  {/if}
{:else if error}
  <div>
    <Error {error}/>
  </div>
{/if}

<style>
    h1.centered, h2.centered {
        text-align: center;
        justify-content: center;
    }

    h1.title {
        margin-bottom: .25rem;
    }

    h1 .pp {
        color: var(--ppColour) !important;
        font-size: smaller;
        border-left: 1px solid var(--dimmed);
        padding-left: .75rem;
        margin-left: .5rem;
    }

    h1 .status {
        font-size: smaller;
    }

    h2.title {
        display: flex;
        margin-bottom: 1rem;
    }

    h2 a {
        border-right: 1px solid var(--dimmed);
        padding: 0 .5rem;
    }

    h2 a:first-of-type {
        padding-left: 0;
    }

    h2 a:last-of-type {
        border-right: none;
    }

    h2 a i {
        color: var(--textColor);
        font-size: smaller;
        position: relative;
        top: -1px;
    }

    h2 a img {
        margin-bottom: 2px;
    }

    .player-role {
        text-align: center;
    }

    @media (max-width: 768px) {
        h2 {
            justify-content: center;
        }
    }
</style>