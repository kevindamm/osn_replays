# Replay loading and database storage

This project aims to crawl and load the Outwitters replay data from
OSN (Outwitters Sports Network) and massage it into a format that is
easier to do analysis and some machine learning from.  Permission has
already been granted to use the data in this way and the maintainer
has since put the OSN project in low-maintenance mode.  The service
is still capable of getting replays of recent games even though the
periodic data dumps have ceased (and it appears the historical ones
have been removed from the server).  There are already 1.2Million
games from various levels of play ability, these can be used as the
basis for an AI, a feature in high demand which is difficult to do
well for this kind of strategy game.

## OSN schema

The OSN schema is nested JSON and contains some superfluous data.  On
its own it would be a poor database format (despite postgres offering
a json column data type) as it would not be possible to index on any
of the contents of the esaped-json-as-a-string nested data (e.g., to
determine if a special unit had been deployed).  Furthermore, it is a
very inefficient means of storing this data -- each json key string
is repeated for each time it occurs (and escaped with \\\" throughout)
-- I suspect we could get 10x or better compressability with even the
simplest data schema.

The json data for a single replay can be obtained with a GET request
to the url http://osn.codepenguin.com/api/getReplay/$gameid where
$gameid can be gathered from a "recent replays" response on the site.
The response payload for a replay is a JSON object having the keys

*   viewResponse (an .hxm filename for the map, no url/path specified)
*   gameState (a string, which when unescaped is the main JSON data)
*   foundRoom (true/false which indicates whether "room" is populated)
*   room (a string, appears to be the same as the game/replay ID)

I will have to check if foundRoom is ever false when OSN has access to
the data, but I suspect not.

When the `gameState` property is unescaped and interpreted as JSON it
also has a `gameState` property.  This is where the actual gameplay
data can be found.  Its properties are:

*   usedSpawns (array of { ix, iy } objects)
*   outcome (integer)
*   gameOverData (another object)
*   settings (metadata for each player)
*   hp\_base0, hp\_base1, ...
*   currentPlayer (first move advantage?)
*   mapName
*   replay

The `replay` property is an array of "frame" objects that define each
move, unit selection, unit spawning, etc. including the initial state
of the board as the first frame, and before each player's turn.


## Schema selection and transformations

We only need to keep some of the properties, and there are aspects of
the serialization format that could be significantly improved.  The
following outline shows selected fields in bold and high-level notes.

*   viewResponse \
    keep children but remove structure
    *   ~~mapName~~ unused
    *   ~~foundRoom~~, ~~room~~ \
        assert == True on response and room == game_id but nopersist either
    *   ~~gameState~~ \
        keep children but remove structure
        *   ~~gcID~~ no need for PII in our analysis
        *   winners, losers \
            have some useful information for current skill level and promotion analysis,
            which could influence policy function or policy selection
        *   ~~usedSpawns~~
        *   ??outcome
    *   settings
        *    color \
             useful for disambiguating side/direction
        *    race
        *    team
        *    [drop rest]
    *   ~~hp_base0~~, ~~hp_base1~~
    *   currentPlayer \
        confirm with color and order of play, may need to invert direction
    *   mapName \
        confirm value with replay_meta
    *   **replay** \
        meat and potatoes, array of Frame objects
        
The properties and transforms for the Frame representation and state transitions of the 
players and environment are complex enough to describe in its own section.

## Frame and Game State representations

