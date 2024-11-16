<script>
    import { round } from '$lib/utils/helper'
    import { getTeamFromTeamManagers } from '$lib/utils/helperFunctions/universalFunctions';

    export let matchup, players, active, ix, displayWeek, expandOverride = false, matchupWeek, leagueTeamManagers, year;

    let home = matchup[0];
    let away = matchup[1];

    let homePointsTotal = 0;
    let homeProjectionTotal = 0;
    let awayPointsTotal = 0;
    let awayProjectionTotal = 0;

    let winning = "home";

    // New function to get the best starters based on points or projections
    const getBestPlayerForPosition = (players, pos, week) => {
        let bestPlayer = null;
        let bestScore = -Infinity;

        // Go through each player, find the highest scorer for the position
        for (const playerId in players) {
            const player = players[playerId];
            if (player.pos === pos && player.wi && player.wi[week]) {
                const score = parseFloat(player.wi[week].p); // Using actual points from the week
                if (score > bestScore) {
                    bestScore = score;
                    bestPlayer = player;
                }
            }
        }
        return bestPlayer;
    };

    const digestStarters = (x, p) => {
        home = matchup[0];
        away = matchup[1];
        home.manager = getTeamFromTeamManagers(leagueTeamManagers, home.roster_id, year);
        away.manager = getTeamFromTeamManagers(leagueTeamManagers, away.roster_id, year);

        // Positions needed for a valid starting lineup (simplified example)
        const requiredPositions = ['QB', 'RB', 'RB', 'WR', 'WR', 'TE', 'FLEX'];

        homePointsTotal = 0;
        homeProjectionTotal = 0;
        awayPointsTotal = 0;
        awayProjectionTotal = 0;

        const localStarters = [];

        // Get the best player for each required position
        requiredPositions.forEach(pos => {
            const homeBest = getBestPlayerForPosition(players, pos, displayWeek);
            const awayBest = getBestPlayerForPosition(players, pos, displayWeek);

            if (homeBest) {
                homePointsTotal += homeBest.wi[displayWeek].p;
                homeProjectionTotal += homeBest.wi[displayWeek].p;
            }

            if (awayBest) {
                awayPointsTotal += awayBest.wi[displayWeek].p;
                awayProjectionTotal += awayBest.wi[displayWeek].p;
            }

            localStarters.push({
                home: homeBest ? digestStarter(homeBest, homePointsTotal) : null,
                away: awayBest ? digestStarter(awayBest, awayPointsTotal) : null
            });
        });

        if (awayPointsTotal < homePointsTotal) winning = "home";
        if (awayPointsTotal > homePointsTotal) winning = "away";
        if (awayPointsTotal == homePointsTotal) winning = "tied";
        starters = localStarters;
    };

    const digestStarter = (starter, points) => {
        if (!starter) {
            return {
                name: "Empty",
                avatar: null,
                pos: null,
                team: null,
                opponent: null,
                projection: 0,
                points: 0,
            };
        }

        let name = starter.pos === "DEF" ? starter.ln : `${starter.fn[0]}. ${starter.ln}`;
        let projection = starter.wi && starter.wi[displayWeek] ? parseFloat(starter.wi[displayWeek].p) : 0;

        return {
            name,
            avatar: starter.pos === "DEF" 
                ? `url(https://sleepercdn.com/images/team_logos/nfl/${starter.team.toLowerCase()}.png)` 
                : `url(https://sleepercdn.com/content/nfl/players/thumb/${starter.id}.jpg)`,
            pos: starter.pos,
            team: starter.t,
            opponent: starter.wi && starter.wi[displayWeek] ? starter.wi[displayWeek].o : null,
            projection,
            points,
        };
    }

    let starters;

    $: digestStarters(ix, players, matchupWeek);

    let el;
    $: top = el?.getBoundingClientRect() ? el?.getBoundingClientRect().top : 0;

    const expandClose = () => {
        if (expandOverride) return;
        active = active == ix ? null : ix;
        setTimeout(() => {
            window.scrollTo({ left: 0, top, behavior: 'smooth' });
        }, 200);
    };

    let innerWidth;

    const calcHeight = () => {
        let multiplier = 73;
        if (innerWidth < 500) {
            multiplier = 72;
        }
        if (innerWidth < 410) {
            multiplier = 71;
        }
        const startersLength = matchupWeek ? home.starters[matchupWeek].length : home.starters.length;
        return startersLength * multiplier + 37;
    }
</script>

<svelte:window bind:innerWidth={innerWidth} />

<style>
    /* Keep your existing styles, no major changes required */
</style>

<div class="matchup">
    <div class="header" on:click={() => expandClose()} bind:this={el}>
        <div class="opponent home{winning == "home" ? " homeGlow" : ""}">
            <img class="avatar" src={home.manager.avatar} alt="home team avatar" />
            <div class="name">{home.manager.name}</div>
            <div class="totalPoints totalPointsR">{round(homePointsTotal)}<div class="totalProjection">{round(homeProjectionTotal)}</div></div>
        </div>
        <img class="divider" src="/{winning}Divider.jpg" alt="divider" />
        <div class="opponent away{winning == "away" ? " awayGlow" : ""}">
            <div class="totalPoints totalPointsL">{round(awayPointsTotal)}<div class="totalProjection">{round(awayProjectionTotal)}</div></div>
            <div class="name">{away.manager.name}</div>
            <img class="avatar" src={away.manager.avatar} alt="away team avatar" />
        </div>
    </div>

    <div class="rosters" style="max-height: {active == ix ? calcHeight() + 'px' : '0'}; {active != ix ? 'border: none' : ''};">
        {#each starters as player}
            <div class="line">
                <div class="player playerHome">
                    <span class="iconAndTeam iconAndTeamHome">
                        {#if player.home.pos}
                            <span class="pos {player.home.pos}">{player.home.pos}</span>
                        {/if}
                        {#if player.home.avatar}
                            <div class="playerAvatar playerInfo" style="background-image: {player.home.avatar}">
                                {#if player.home.team && player.home.pos != "DEF"}
                                    <img src="https://sleepercdn.com/images/team_logos/nfl/{player.home.team.toLowerCase()}.png" class="teamLogo teamHomeLogo" alt="team logo"/>
                                {/if}
                            </div>
                        {/if}
                    </span>
                    <div class="nameHolder nameHolderL{player.home.name == 'Empty'? ' playerEmpty' : ''}">
                        <span class="playerInfo playerName playerNameHome">{player.home.name}</span>
                        {#if player.home.team}
                            {#if player.home.opponent}
                                <div class="playerTeam">{player.home.pos != "DEF" ? `${player.home.team}` : ""} vs {player.home.opponent}</div>
                            {:else}
                                <div class="playerTeam">{player.home.pos != "DEF" ? player.home.team : ""}</div>
                            {/if}
                        {/if}
                    </div>
                    <span class="points pointsR">{round(player.home.points)}<div class="totalProjection">{round(player.home.projection)}</div></span>
                </div>

                <div class="dividerLine" />

                <div class="player playerAway">
                    <span class="iconAndTeam iconAndTeamAway">
                        {#if player.away.pos}
                            <span class="pos {player.away.pos}">{player.away.pos}</span>
                        {/if}
                        {#if player.away.avatar}
                            <div class="playerAvatar playerInfo" style="background-image: {player.away.avatar}">
                                {#if player.away.team && player.away.pos != "DEF"}
                                    <img src="https://sleepercdn.com/images/team_logos/nfl/{player.away.team.toLowerCase()}.png" class="teamLogo teamAwayLogo" alt="team logo"/>
                                {/if}
                            </div>
                        {/if}
                    </span>
                    <div class="nameHolder nameHolderL{player.away.name == 'Empty'? ' playerEmpty' : ''}">
                        <span class="playerInfo playerName playerNameAway">{player.away.name}</span>
                        {#if player.away.team}
                            {#if player.away.opponent}
                                <div class="playerTeam">{player.away.pos != "DEF" ? `${player.away.team}` : ""} vs {player.away.opponent}</div>
                            {:else}
                                <div class="playerTeam">{player.away.pos != "DEF" ? player.away.team : ""}</div>
                            {/if}
                        {/if}
                    </div>
                    <span class="points pointsL">{round(player.away.points)}<div class="totalProjection">{round(player.away.projection)}</div></span>
                </div>
            </div>
        {/each}
    </div>
</div>
