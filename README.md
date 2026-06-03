<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Drive Mad Remake</title>
    <style>
        body {
            margin: 0;
            padding: 0;
            background: #1a1a1a;
            color: #fff;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            overflow: hidden;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            height: 100vh;
        }
        #ui {
            position: absolute;
            top: 20px;
            text-align: center;
            z-index: 10;
            pointer-events: none;
        }
        h1 {
            margin: 0 0 5px 0;
            font-size: 24px;
            text-transform: uppercase;
            letter-spacing: 2px;
        }
        p {
            margin: 0;
            font-size: 14px;
            color: #aaa;
        }
        canvas {
            box-shadow: 0 10px 30px rgba(0,0,0,0.5);
            border-radius: 8px;
            background: linear-gradient(to bottom, #87CEEB, #E0F6FF);
        }
    </style>
    <!-- Include Matter.js Physics Engine via CDN -->
    <script src="https://cloudflare.com"></script>
</head>
<body>

    <div id="ui">
        <h1>Drive Mad 2D Remake</h1>
        <p>Controls: D / Right Arrow = Accelerate | A / Left Arrow = Reverse | R = Restart</p>
    </div>

    <script>
        // Matter.js Module Aliases
        const { Engine, Render, Runner, Bodies, Composite, Constraint, Body, Vector } = Matter;

        // Create Engine and World
        const engine = Engine.create();
        const world = engine.world;
        world.gravity.y = 1.2; // Realistic gravity simulation

        // Setup Renderer
        const canvasWidth = 1000;
        const canvasHeight = 600;
        const render = Render.create({
            element: document.body,
            engine: engine,
            options: {
                width: canvasWidth,
                height: canvasHeight,
                wireframes: false, // Set to false to see solid colours
                background: 'transparent'
            }
        });

        Render.run(render);
        const runner = Runner.create();
        Runner.run(runner, engine);

        // Game State variables
        let car, wheelA, wheelB;
        let keys = {};

        // --- Build Terrain / Environment ---
        function createTerrain() {
            // Flat starting zone
            Composite.add(world, Bodies.rectangle(200, 550, 500, 40, { isStatic: true, render: { fillStyle: '#4CAF50' } }));
            
            // Step Obstacle 1
            Composite.add(world, Bodies.rectangle(550, 530, 100, 80, { isStatic: true, render: { fillStyle: '#8B4513' } }));
            
            // Bridge gap segments
            Composite.add(world, Bodies.rectangle(750, 550, 150, 40, { isStatic: true, render: { fillStyle: '#4CAF50' } }));
            
            // Massive jump ramp
            Composite.add(world, Bodies.rectangle(1000, 510, 200, 40, { 
                isStatic: true, 
                angle: -0.3, 
                render: { fillStyle: '#FF9800' } 
            }));

            // Landing zone
            Composite.add(world, Bodies.rectangle(1400, 580, 400, 40, { isStatic: true, render: { fillStyle: '#4CAF50' } }));
            
            // Finish Line marker
            Composite.add(world, Bodies.rectangle(1550, 510, 10, 100, { isStatic: true, render: { fillStyle: '#000000' } }));
        }

        // --- Spawn Monster Truck ---
        function spawnVehicle(x, y) {
            // Main chassis body
            car = Bodies.rectangle(x, y, 110, 30, {
                collisionFilter: { group: -1 },
                render: { fillStyle: '#E91E63' }
            });

            // Top cabin styling
            const cabin = Bodies.rectangle(x - 10, y - 25, 60, 30, {
                collisionFilter: { group: -1 },
                render: { fillStyle: '#C2185B' }
            });

            // Merge chassis segments
            const carBody = Body.create({
                parts: [car, cabin],
                collisionFilter: { group: -1 }
            });

            // High-traction monster wheels
            const wheelOptions = {
                friction: 0.9,
                density: 0.01,
                collisionFilter: { group: -1 },
                render: { fillStyle: '#333333' }
            };
            wheelA = Bodies.circle(x - 40, y + 25, 28, wheelOptions);
            wheelB = Bodies.circle(x + 40, y + 25, 28, wheelOptions);

            // Suspension Constraints (Springs)
            const springOptions = {
                stiffness: 0.13,
                damping: 0.08,
                render: { visible: true, strokeStyle: '#555', lineWidth: 4 }
            };

            const springA1 = Constraint.create({ bodyA: carBody, pointA: { x: -40, y: 15 }, bodyB: wheelA, ...springOptions });
            const springA2 = Constraint.create({ bodyA: carBody, pointA: { x: -40, y: -5 }, bodyB: wheelA, ...springOptions });
            const springB1 = Constraint.create({ bodyA: carBody, pointA: { x: 40, y: 15 }, bodyB: wheelB, ...springOptions });
            const springB2 = Constraint.create({ bodyA: carBody, pointA: { x: 40, y: -5 }, bodyB: wheelB, ...springOptions });

            Composite.add(world, [carBody, wheelA, wheelB, springA1, springA2, springB1, springB2]);
            car = carBody; // Reference point for the camera tracker
        }

        // --- Initialize Game Elements ---
        function initGame() {
            Composite.clear(world, false);
            createTerrain();
            spawnVehicle(150, 450);
        }

        // --- Input Controls ---
        window.addEventListener('keydown', e => {
            keys[e.key.toLowerCase()] = true;
            keys[e.code] = true;
            if (e.key.toLowerCase() === 'r') initGame();
        });
        window.addEventListener('keyup', e => {
            keys[e.key.toLowerCase()] = false;
            keys[e.code] = false;
        });

        // --- Game Logic & Camera Tracking Loop ---
        Matter.Events.on(engine, 'afterUpdate', () => {
            if (!car) return;

            const speed = 0.28; // Torque power
            
            // Drive Forward
            if (keys['d'] || keys['arrowright']) {
                Body.setAngularVelocity(wheelA, speed);
                Body.setAngularVelocity(wheelB, speed);
            }
            // Drive Reverse / Brake
            if (keys['a'] || keys['arrowleft']) {
                Body.setAngularVelocity(wheelA, -speed);
                Body.setAngularVelocity(wheelB, -speed);
            }

            // Smooth Camera Follow Engine
            const targetX = car.position.x - canvasWidth / 3;
            const targetY = car.position.y - canvasHeight / 1.5;
            
            // Clamp view height bounds
            const boundedY = Math.max(-200, Math.min(100, targetY));

            Render.lookAt(render, {
                min: { x: targetX, y: boundedY },
                max: { x: targetX + canvasWidth, y: boundedY + canvasHeight }
            });
        });

        // Run Setup on Boot
        initGame();
    </script>
</body>
</html>
