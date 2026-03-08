Yes — any web-based game can be an XBlock. Here's the full picture:

How It Works
An XBlock is just a wrapper. The game lives inside it, and the XBlock handles:

Saving progress (via XBlock fields → MongoDB)
Awarding XP (publish xp_earned event)
Reporting grades (publish grade event → gradebook)
Studio config (instructor sets difficulty, time limits, etc.)
Game Types That Work
1. Built directly into the XBlock (simplest)
The game JS runs inside the XBlock's own HTML fragment:

my-game-xblock/
├── my_game/
│   ├── static/
│   │   ├── html/student.html   ← game canvas lives here
│   │   ├── js/game.js          ← Phaser, vanilla canvas, etc.
│   │   └── css/game.css

The game JS has direct access to runtime.handlerUrl() — no postMessage needed.

2. iframe-embedded (like PhETSim pattern)
Game is hosted separately (or as a static file), embedded in an iframe. Communicates via postMessage:

XBlock JS ←── postMessage ──── Game iframe
              (score, events)

Works for: Unity WebGL, Godot Web, any game hosted on a URL.

3. Full 3D / engine-based games
Engine	Export Format	Works as XBlock?
Phaser.js	HTML5 Canvas	✅ Native (built in)
Unity	WebGL	✅ iframe
Godot	Web export	✅ iframe
Babylon.js	HTML5	✅ Native
Three.js	HTML5	✅ Native
PixiJS	HTML5 Canvas	✅ Native
p5.js	HTML5 Canvas	✅ Native
Minimal Game XBlock Example (Phaser.js)
# game_xblock.py
class GameXBlock(XBlock):
    high_score = Integer(default=0, scope=Scope.user_state)
    completed  = Boolean(default=False, scope=Scope.user_state)

    def student_view(self, context=None):
        frag = Fragment(self.resource_string("static/html/game.html"))
        frag.add_javascript(self.resource_string("static/js/phaser.min.js"))
        frag.add_javascript(self.resource_string("static/js/game.js"))
        frag.initialize_js('GameXBlock', {'high_score': self.high_score})
        return frag

    @XBlock.json_handler
    def save_score(self, data, suffix=''):
        score = data.get('score', 0)
        if score > self.high_score:
            self.high_score = score
        if score >= 1000 and not self.completed:
            self.completed = True
            self.runtime.publish(self, 'xp_earned', {'xp': 100, 'action': 'game_won'})
            self.runtime.publish(self, 'grade', {'value': 1.0, 'max_value': 1.0})
        return {'success': True, 'high_score': self.high_score}

// game.js — Phaser game calls XBlock handler on game events
function GameXBlock(runtime, element, initArgs) {
    var saveUrl = runtime.handlerUrl(element, 'save_score');

    var config = {
        type: Phaser.AUTO,
        parent: element.querySelector('#game-container'),
        scene: {
            update: function() { /* game logic */ },
            create: function() { /* setup */ }
        }
    };

    var game = new Phaser.Game(config);

    // Call this when player wins/scores
    window.reportScore = function(score) {
        fetch(saveUrl, {
            method: 'POST',
            headers: {'Content-Type': 'application/json',
                      'X-CSRFToken': getCsrfToken()},
            body: JSON.stringify({score: score})
        });
    };
}

What You Can Track Per Game
Data	XBlock Field	Where stored
High score	Integer(scope=Scope.user_state)	MongoDB
Level reached	Integer(scope=Scope.user_state)	MongoDB
Attempts	Integer(scope=Scope.user_state)	MongoDB
Time played	Integer(scope=Scope.user_state)	MongoDB
Completion	Boolean(scope=Scope.user_state)	MongoDB
Saved game state	Dict(scope=Scope.user_state)	MongoDB
The Only Constraints
Must be web-based — runs in a browser (no native apps)
No persistent server needed — game state goes through XBlock handlers, not a separate game server (unless you do the sidecar pattern like Multiplayer XBlock)
iframes need postMessage to talk back to the XBlock
So yes — puzzle games, platformers, quizzes, simulations, strategy games, multiplayer games — all work. The Multiplayer XBlock we already planned is essentially a real-time competitive game engine.