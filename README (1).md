<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <style>
        canvas {
            position: absolute;
            top: 50%;
            left: 50%;
            width: 640px;
            height: 640px;
            margin: -320px 0 0 -320px;
        }
    </style>
</head>
<body>
    <canvas></canvas>
    <script>
        'use strict';
        var canvas = document.querySelector('canvas');
        canvas.width = 640;
        canvas.height = 640;

        var g = canvas.getContext('2d');

        var right = { x: 1, y: 0 };
        var down = { x: 0, y: 1 };
        var left = { x: -1, y: 0 };

        var EMPTY = -1;
        var BORDER = -2;

        var fallingShape;
        var nextShape;
        var nRows = 18;
        var nCols = 12;
        var blockSize = 30;
        var topMargin = 50;
        var leftMargin = 20;
        var scoreX = 400;
        var scoreY = 330;
        var titleX = 130;
        var titleY = 160;
        var clickX = 120;
        var clickY = 400;
        var previewCenterX = 467;
        var previewCenterY = 97;
        var mainFont = 'bold 48px monospace';
        var smallFont = 'bold 18px monospace';
        var colors = ['green', 'red', 'blue', 'purple', 'orange', 'blueviolet', 'magenta'];
        var gridRect = { x: 46, y: 47, w: 308, h: 517 };
        var previewRect = { x: 387, y: 47, w: 200, h: 200 };
        var outerRect = { x: 5, y: 5, w: 630, h: 630 };
        var squareBorder = 'white';
        var titlebgColor = 'white';
        var textColor = 'black';
        var bgColor = '#DDEEFF';
        var gridColor = '#BECFEA';
        var gridBorderColor = '#7788AA';
        var largeStroke = 5;
        var smallStroke = 2;

        var fallingShapeRow;
        var fallingShapeCol;

        var keyDown = false;
        var fastDown = false;

        var grid = [];
        var scoreboard = new Scoreboard();

        addEventListener('keydown', function (event) {
            if (!keyDown) {
                keyDown = true;

                if (scoreboard.isGameOver()) return;

                switch (event.key) {
                    case 'w':
                    case 'ArrowUp':
                        if (canRotate(fallingShape)) rotate(fallingShape);
                        break;
                    case 'a':
                    case 'ArrowLeft':
                        if (canMove(fallingShape, left)) move(left);
                        break;
                    case 'd':
                    case 'ArrowRight':
                        if (canMove(fallingShape, right)) move(right);
                        break;
                    case 's':
                    case 'ArrowDown':
                        if (!fastDown) {
                            fastDown = true;
                            while (canMove(fallingShape, down)) {
                                move(down);
                                draw();
                            }
                            shapeHasLanded();
                        }
                        break;
                }
                draw();
            }
        });

        addEventListener('click', function () {
            if (scoreboard.isGameOver()) {
                startNewGame();
            }
        });

        addEventListener('keyup', function () {
            keyDown = false;
            fastDown = false;
        });

        function canRotate(s) {
            if (s === Shapes.Square) return false;

            var pos = s.pos.map(function (row) {
                return row.slice();
            });

            pos.forEach(function (row) {
                var tmp = row[0];
                row[0] = row[1];
                row[1] = -tmp;
            });

            return pos.every(function (p) {
                var newCol = fallingShapeCol + p[0];
                var newRow = fallingShapeRow + p[1];
                return grid[newRow][newCol] === EMPTY;
            });
        }

        function rotate(s) {
            if (s === Shapes.Square) return;

            s.pos.forEach(function (row) {
                var tmp = row[0];
                row[0] = row[1];
                row[1] = -tmp;
            });
        }

        function move(dir) {
            fallingShapeRow += dir.y;
            fallingShapeCol += dir.x;
        }

        function canMove(s, dir) {
            return s.pos.every(function (p) {
                var newCol = fallingShapeCol + dir.x + p[0];
                var newRow = fallingShapeRow + dir.y + p[1];
                return grid[newRow][newCol] === EMPTY;
            });
        }

        function shapeHasLanded() {
            addShape(fallingShape);
            if (fallingShapeRow < 2) {
                scoreboard.setGameOver();
                scoreboard.setTopscore();
            } else {
                scoreboard.addLines(removeLines());
            }
            selectShape();
        }

        function removeLines() {
            var count = 0;
            for (var r = 0; r < nRows - 1; r++) {
                for (var c = 1; c < nCols - 1; c++) {
                    if (grid[r][c] === EMPTY) break;
                    if (c === nCols - 2) {
                        count++;
                        removeLine(r);
                    }
                }
            }
            return count;
        }

        function removeLine(line) {
            for (var c = 0; c < nCols; c++) grid[line][c] = EMPTY;

            for (var c = 0; c < nCols; c++) {
                for (var r = line; r > 0; r--) grid[r][c] = grid[r - 1][c];
            }
        }

        function addShape(s) {
            s.pos.forEach(function (p) {
                grid[fallingShapeRow + p[1]][fallingShapeCol + p[0]] = s.ordinal;
            });
        }

        function Shape(shape, o) {
            this.shape = shape;
            this.pos = this.reset();
            this.ordinal = o;
        }

        var Shapes = {
            ZShape: [[0, -1], [0, 0], [-1, 0], [-1, 1]],
            SShape: [[0, -1], [0, 0], [1, 0], [1, 1]],
            IShape: [[0, -1], [0, 0], [0, 1], [0, 2]],
            TShape: [[-1, 0], [0, 0], [1, 0], [0, 1]],
            Square: [[0, 0], [1, 0], [0, 1], [1, 1]],
            LShape: [[-1, -1], [0, -1], [0, 0], [0, 1]],
            JShape: [[1, -1], [0, -1], [0, 0], [0, 1]]
        };

        function getRandomShape() {
            var keys = Object.keys(Shapes);
            var ord = Math.floor(Math.random() * keys.length);
            var shape = Shapes[keys[ord]];
            return new Shape(shape, ord);
        }

        Shape.prototype.reset = function () {
            this.pos = this.shape.map(function (row) {
                return row.slice();
            });
            return this.pos;
        };

        function selectShape() {
            fallingShapeRow = 1;
            fallingShapeCol = 5;
            fallingShape = nextShape;
            nextShape = getRandomShape();
            if (fallingShape != null) {
                fallingShape.reset();
            }
        }

        function Scoreboard() {
            this.MAXLEVEL = 9;
            var level = 0;
            var lines = 0;
            var score = 0;
            var topscore = 0;
            var gameOver = true;

            this.reset = function () {
                this.setTopscore();
                level = lines = score = 0;
                gameOver = false;
            }

            this.setGameOver = function () {
                gameOver = true;
            }

            this.isGameOver = function () {
                return gameOver;
            }

            this.setTopscore = function () {
                if (score > topscore) {
                    topscore = score;
                }
            }

            this.getTopscore = function () {
                return topscore;
            }

            this.getSpeed = function () {
                switch (level) {
                    case 0: return 700;
                    case 1: return 600;
                    case 2: return 500;
                    case 3: return 400;
                    case 4: return 300;
                    case 5: return 250;
                    case 6: return 200;
                    case 7: return 150;
                    case 8: return 100;
                    default: return 100;
                }
            }

            this.addLines = function (l) {
                lines += l;
                level = Math.floor(lines / 10);
                score += l * (level + 1);
            }

            this.getLevel = function () {
                return level;
            }

            this.getLines = function () {
                return lines;
            }

            this.getScore = function () {
                return score;
            }
        }

        function draw() {
            g.fillStyle = bgColor;
            g.fillRect(0, 0, canvas.width, canvas.height);
            drawGrid();
            drawShapes();
            drawScoreboard();
        }

        function drawGrid() {
            g.fillStyle = gridColor;
            g.strokeStyle = gridBorderColor;
            g.lineWidth = smallStroke;

            for (var r = 0; r < nRows; r++) {
                for (var c = 0; c < nCols; c++) {
                    g.beginPath();
                    g.rect(leftMargin + c * blockSize, topMargin + r * blockSize, blockSize, blockSize);
                    g.stroke();
                    if (grid[r][c] === EMPTY) {
                        g.fillStyle = bgColor;
                        g.fill();
                    } else {
                        g.fillStyle = colors[grid[r][c]];
                        g.fill();
                    }
                }
            }
        }

        function drawShapes() {
            fallingShape.pos.forEach(function (row) {
                g.fillStyle = colors[fallingShape.ordinal];
                g.fillRect(leftMargin + (fallingShapeCol + row[0]) * blockSize,
                    topMargin + (fallingShapeRow + row[1]) * blockSize,
                    blockSize, blockSize);
            });

            nextShape.pos.forEach(function (row) {
                g.fillStyle = colors[nextShape.ordinal];
                g.fillRect(previewCenterX + (row[0]) * blockSize,
                    previewCenterY + (row[1]) * blockSize,
                    blockSize, blockSize);
            });
        }

        function drawScoreboard() {
            g.fillStyle = titlebgColor;
            g.font = mainFont;
            g.fillText('TETRIS', titleX, titleY);

            g.fillStyle = textColor;
            g.font = smallFont;
            g.fillText('Lines: ' + scoreboard.getLines(), scoreX, scoreY);
            g.fillText('Level: ' + scoreboard.getLevel(), scoreX, scoreY + 30);
            g.fillText('Score: ' + scoreboard.getScore(), scoreX, scoreY + 60);
            g.fillText('Topscore: ' + scoreboard.getTopscore(), scoreX, scoreY + 90);

            if (scoreboard.isGameOver()) {
                g.fillText('GAME OVER', 170, 300);
                g.fillText('CLICK TO START', clickX, clickY);
            }
        }

        function startNewGame() {
            grid = [];
            for (var r = 0; r < nRows; r++) {
                grid[r] = [];
                for (var c = 0; c < nCols; c++) {
                    grid[r][c] = EMPTY;
                }
            }

            scoreboard.reset();
            selectShape();
            draw();
            setInterval(function () {
                if (!scoreboard.isGameOver()) {
                    if (canMove(fallingShape, down)) {
                        move(down);
                    } else {
                        shapeHasLanded();
                    }
                    draw();
                }
            }, scoreboard.getSpeed());
        }

        startNewGame();
    </script>
</body>
</html>
