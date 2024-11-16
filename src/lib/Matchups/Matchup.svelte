<script>
    import { round } from '$lib/utils/helper';
    import { getTeamFromTeamManagers } from '$lib/utils/helperFunctions/universalFunctions';

    export let matchup, players, active, ix, displayWeek, expandOverride = false, matchupWeek, leagueTeamManagers, year;

    let home = matchup[0];
    let away = matchup[1];

    let homePointsTotal = 0;
    let homeProjectionTotal = 0;
    let awayPointsTotal = 0;
    let awayProjectionTotal = 0;

    let winning = "home";

    // Define position requirements (customize as needed)
    const positionRequirements = {
        QB: 1,
        RB: 2,
        WR: 2,
        TE: 1,
        FLEX: 2, // FLEX can be RB, WR, or TE
        K: 1,
        DST: 1
    };

    const digestStarters = (x, p) => {
        home = matchup[0];
        away = matchup[1];
        home.manager = getTeamFromTeamManagers(leagueTeamManagers, home.roster_id, year);
        away.manager = getTeamFromTeamManagers(leagueTeamManagers, away.roster_id, year);

        const homeStarters = matchupWeek ? home.starters[matchupWeek] : home.starters;
        const awayStarters = matchupWeek ? away.starters[matchupWeek] : away.starters;
        const homePoints = matchupWeek ? home.points[matchupWeek] : home.points;
        const awayPoints = matchupWeek ? away.points[matchupWeek] : away.points;

        homePointsTotal = 0;
        homeProjectionTotal = 0;
        awayPointsTotal = 0;
        awayProjectionTotal = 0;

        // Sort and filter players by position
        const sortedHomePlayers = getSortedPlayers(home.starters, home.projections, displayWeek);
        const sortedAwayPlayers = getSortedPlayers(away.starters, away.projections, displayWeek);

        const localStarters = [];

        // Assign players to positions based on position requirements
        const homeLineup = assignPositionsToLineup(sortedHomePlayers, positionRequirements, homePoints);
        const awayLineup = assignPositionsToLineup(sortedAwayPlayers, positionRequirements, awayPoints);

        homePointsTotal = homeLineup.totalPoints;
        awayPointsTotal = awayLineup.totalPoints;

        homeProjectionTotal = homeLineup.totalProjection;
        awayProjectionTotal = awayLineup.totalProjection;

        for (let i = 0; i < homeLineup.players.length; i++) {
            const homeStarter = homeLineup.players[i];
            const awayStarter = awayLineup.players[i];

            localStarters.push({
                home: digestStarter(homeStarter.id, homeStarter.points),
                away: digestStarter(awayStarter.id, awayStarter.points)
            });
        }

        // Determine the winner
        if (awayPointsTotal < homePointsTotal) winning = "home";
        if (awayPointsTotal > homePointsTotal) winning = "away";
        if (awayPointsTotal == homePointsTotal) winning = "tied";

        starters = localStarters;
    };

    // Helper function to sort players by projection and filter by position
    const getSortedPlayers = (starters, projections, week) => {
        return starters.map(starterId => {
            const player = players[starterId];
            return {
                id: starterId,
                projection: player.wi && player.wi[week] ? parseFloat(player.wi[week].p) : 0,
                pos: player.pos
            };
        }).sort((a, b) => b.projection - a.projection); // Sort by projection in descending order
    };

    // Helper function to assign players to positions based on lineup requirements
    const assignPositionsToLineup = (sortedPlayers, positionRequirements, pointsArray) => {
        let positionCount = {
            QB: 0,
            RB: 0,
            WR: 0,
            TE: 0,
            FLEX: 0,
            K: 0,
            DST: 0
        };

        let playersByPosition = {
            QB: [],
            RB: [],
            WR: [],
            TE: [],
            FLEX: [],
            K: [],
            DST: []
        };

        // Distribute sorted players into their positions
        sortedPlayers.forEach(player => {
            if (positionCount[player.pos] < positionRequirements[player.pos]) {
                playersByPosition[player.pos].push(player);
                positionCount[player.pos]++;
            } else if (player.pos === 'RB' || player.pos === 'WR' || player.pos === 'TE') {
                // For FLEX positions, allow RB/WR/TE to be flexible
                if (positionCount.FLEX < positionRequirements.FLEX) {
                    playersByPosition.FLEX.push(player);
                    positionCount.FLEX++;
                }
            }
        });

        // Now ensure that each position is filled up to the required number
        let selectedPlayers = [];

        Object.keys(positionRequirements).forEach(position => {
            const requiredCount = positionRequirements[position];
            const selected = playersByPosition[position].slice(0, requiredCount);

            // Add selected players to the final lineup
            selectedPlayers = selectedPlayers.concat(selected);
        });

        // Calculate total points and projections
        let totalPoints = 0;
        let totalProjection = 0;
        selectedPlayers.forEach((player, index) => {
            totalPoints += pointsArray[index] || 0;
            totalProjection += player.projection || 0;
        });

        return {
            players: selectedPlayers,
            totalPoints: totalPoints,
            totalProjection: totalProjection
        };
    };

    const digestStarter = (starterId, points) => {
        if (!starterId || starterId == 0) {
            return {
                name: "Empty",
                avatar: null,
                poss: null,
                team: null,
                opponent: null,
                projection: 0,
                points: 0,
            };
        }

        const player = players[starterId];
        let name = player.pos == "DEF" ? player.ln : `${player.fn[0]}. ${player.ln}`;
        let projection = 0;

        if (player.wi && player.wi[displayWeek]) {
            projection = parseFloat(player.wi[displayWeek].p);
        }

        return {
            name,
            avatar: player.pos == "DEF" ? `background-image: url(https://sleepercdn.com/images/team_logos/nfl/${starterId.toLowerCase()}.png)` : `background-image: url(https://sleepercdn.com/content/nfl/players/thumb/${starterId}.jpg), url(https://sleepercdn.com/images/v2/icons/player_default.webp)`,
            pos: player.pos,
            team: player.t,
            opponent: player.wi && player.wi[displayWeek] ? player.wi[displayWeek].o : null,
            projection,
            points,
        };
    };

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
        if (innerWidth < 400) {
            multiplier = 60;
        }
        return innerWidth < 400 ? '300' : `${starters?.length * multiplier}`;
    };

    const calcWidth = () => {
        return '100%';
    };

</script>
