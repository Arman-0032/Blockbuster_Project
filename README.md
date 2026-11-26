# Blockbuster_Project ,
#include <SFML/Graphics.hpp>
#include <optional>
#include <iostream>
#include <fstream>
#include <string>
#include <vector>
#include <cmath>
#include <cstdint>
#include <algorithm>
#include <cstdlib>
#include <ctime>

// =====================================================================
//  CONSTANTS
// =====================================================================
const bool DEBUG = false;                   // true = print events to the console

const int WINDOW_W = 600;
const int WINDOW_H = 700;
const int HUD_H = 50;

const int ROWS = 8;
const int COLS = 10;
const float BRICK_W = 60;
const float BRICK_H = 25;
const float BRICKS_Y = 100;

const float PADDLE_W = 100;
const float PADDLE_H = 14;
const float PADDLE_Y = 650;
const float PADDLE_SPEED = 8;

const float BALL_SIZE = 12;
const float BALL_START_Y = 638;


const float POWERUP_SIZE = 20;
const float POWERUP_SPEED = 3;                
const int   WIDE_FRAMES = 600;                 
const float WIDE_W = 150;

enum GameState { MENU, PLAYING, GAMEOVER, WIN };

// =====================================================================
//  THEMES AND VISUAL TYPES (animation only, no game rules in here)
// =====================================================================
sf::Color rgb(int r, int g, int b)
{
    return sf::Color((std::uint8_t)r, (std::uint8_t)g, (std::uint8_t)b);
}

struct Theme
{
    const char* name;
    sf::Color bg, hud, brick[4], paddle, ball, accent, text, star;
    bool glow;                              // additive glow and scanlines
};

const int THEME_COUNT = 3;
const Theme themes[THEME_COUNT] = {
    { "Neon arcade", rgb(13, 2, 33), rgb(36, 0, 70),
      { rgb(0, 0, 0), rgb(5, 217, 232), rgb(249, 200, 14), rgb(255, 42, 109) },
      rgb(5, 217, 232), rgb(255, 255, 255), rgb(255, 70, 170), rgb(255, 255, 255),
      rgb(123, 47, 247), true },

    { "Deep space", rgb(11, 16, 38), rgb(27, 33, 80),
      { rgb(0, 0, 0), rgb(76, 201, 240), rgb(123, 108, 246), rgb(247, 37, 133) },
      rgb(224, 225, 221), rgb(255, 214, 10), rgb(76, 201, 240), rgb(255, 255, 255),
      rgb(141, 153, 174), true },

    { "Candy pop", rgb(255, 240, 246), rgb(255, 200, 221),
      { rgb(0, 0, 0), rgb(162, 210, 255), rgb(255, 175, 204), rgb(255, 93, 143) },
      rgb(140, 82, 255), rgb(60, 20, 120), rgb(255, 93, 143), rgb(80, 30, 90),
      rgb(255, 214, 231), false }
};

struct Particle { float x, y, vx, vy, life, maxLife, size; sf::Color color; };
struct Popup    { float x, y, life; std::string text; sf::Color color; };
struct Star     { float x, y, speed, size, phase; };
struct BrickEvent { int r, c, before; bool broke; };   // filled by hitBricks

float frand(float a, float b)
{
    return a + (b - a) * (rand() / (float)RAND_MAX);
}

sf::Color withAlpha(sf::Color c, float a)
{
    c.a = (std::uint8_t)std::clamp(a, 0.f, 255.f);
    return c;
}

sf::Color mixColor(sf::Color a, sf::Color b, float t)
{
    t = std::clamp(t, 0.f, 1.f);
    return sf::Color((std::uint8_t)(a.r + (b.r - a.r) * t),
                     (std::uint8_t)(a.g + (b.g - a.g) * t),
                     (std::uint8_t)(a.b + (b.b - a.b) * t));
}

void spawnBurst(std::vector<Particle>& ps, float x, float y, sf::Color col, int n, float speed)
{
    for (int i = 0; i < n; i++)
    {
        float angle = frand(0.f, 6.2832f);
        float sp = frand(0.3f, 1.f) * speed;
        float life = frand(0.5f, 1.0f);
        ps.push_back({ x, y, std::cos(angle) * sp, std::sin(angle) * sp - speed * 0.3f,
                       life, life, frand(3.f, 7.f), col });
    }
}

// =====================================================================
//  GAME DATA AND FUNCTIONS (the rules, same as the spec)
// =====================================================================
int bricks[ROWS][COLS];                     // hits left for each brick
std::vector<BrickEvent> brickEvents;        // what hitBricks hit (for effects)

bool loadLevel()
{
    std::ifstream fin("level.txt");
    if (!fin) return false;

    int rows, cols;
    fin >> rows >> cols;

    for (int r = 0; r < ROWS; r++)
        for (int c = 0; c < COLS; c++)
            fin >> bricks[r][c];

    return true;
}


bool hitBricks(float ballX, float ballY, int &score)
{
    float cornersX[4] = { ballX, ballX + BALL_SIZE - 1, ballX,ballX + BALL_SIZE - 1 };
    float cornersY[4] = { ballY, ballY,ballY + BALL_SIZE - 1, ballY + BALL_SIZE - 1 };

    bool touched[ROWS][COLS] = {};
    bool anyHit = false;

    for (int i = 0; i < 4; i++)
    {
        float cx = cornersX[i];
        float cy = cornersY[i];

        if (cy < BRICKS_Y || cy >= BRICKS_Y + ROWS * BRICK_H)
            continue;
        if (cx < 0 || cx >= WINDOW_W) 
            continue;

        int r = (int)((cy - BRICKS_Y) / BRICK_H);
        int c = (int)(cx / BRICK_W);

        if (bricks[r][c] == 0) 
            continue;
        if (touched[r][c]) 
            continue;

        touched[r][c] = true;
        int before = bricks[r][c];
        bricks[r][c]--;
        anyHit = true;

        bool broke = (bricks[r][c] == 0);
        if (broke)
            score += 50;
        else
            score += 10;
        brickEvents.push_back({ r, c, before, broke });
    }
    return anyHit;
}

int countBricks()
{
    int n = 0;
    for (int r = 0; r < ROWS; r++)
        for (int c = 0; c < COLS; c++)
            if (bricks[r][c] > 0)
                n++;
    return n;
}
bool  powerActive = false;
float powerX = 0, powerY = 0;       
char  powerType = '+';              
void spawnPowerUp(float brickCenterX, float brickCenterY)
{
    if (powerActive) 
        return;
    if (rand() % 5 != 0) 
        return;
    powerActive = true;
    powerX = brickCenterX - POWERUP_SIZE / 2;
    powerY = brickCenterY - POWERUP_SIZE / 2;
    powerType = (rand() % 2 == 0) ? 'W' : '+';
}
char updatePowerUp(float paddleX, float paddleW)
{
    if (!powerActive) return 0;

    powerY += POWERUP_SPEED;

    if (powerX < paddleX + paddleW && powerX + POWERUP_SIZE > paddleX && powerY < PADDLE_Y + PADDLE_H && powerY + POWERUP_SIZE > PADDLE_Y)
    {
        powerActive = false;
        return powerType;
    }

    if (powerY > WINDOW_H)
        powerActive = false;

    return 0;
}

void saveGame(int score, int lives)
{
    std::ofstream fout("savegame.txt");
    fout << score << " " << lives << "\n";

    for (int r = 0; r < ROWS; r++)
    {
        for (int c = 0; c < COLS; c++)
            fout << bricks[r][c] << " ";
        fout << "\n";
    }
}

bool loadGame(int &score, int &lives)
{
    std::ifstream fin("savegame.txt");
    if (!fin) 
        return false;
    int s, l;
    if (!(fin >> s >> l)) 
        return false;
    if (l <= 0) 
        return false;
    int temp[ROWS][COLS];
    for (int r = 0; r < ROWS; r++)
        for (int c = 0; c < COLS; c++)
            if (!(fin >> temp[r][c])) 
                return false;
    score = s;
    lives = l;
    for (int r = 0; r < ROWS; r++)
        for (int c = 0; c < COLS; c++)
            bricks[r][c] = temp[r][c];
    return true;
}

void clearSave()
{
    std::ofstream fout("savegame.txt");
}

int main()
{
    srand((unsigned)time(0));

    sf::RenderWindow window(sf::VideoMode({ WINDOW_W, WINDOW_H }), "Block Buster");
    window.setFramerateLimit(60);

    sf::View view = window.getDefaultView();
    sf::Clock clock;

    sf::RenderStates additive;
    additive.blendMode = sf::BlendAdd;

    GameState state = MENU;
    int themeIndex = 0;
    sf::RectangleShape rectShape({ 10.f, 10.f });
    sf::CircleShape circleShape(10.f);
    sf::Font font;
    if (!font.openFromFile("arial.ttf"))
        std::cout << "Could not load arial.ttf (copy it next to level.txt)\n";
    sf::Text text(font, "", 24);
    auto placeText = [&](sf::Text& t, const std::string& s, int size, float cx, float y)
    {
        t.setString(s);
        t.setCharacterSize(size);
        sf::FloatRect b = t.getLocalBounds();
        t.setOrigin({ b.position.x + b.size.x / 2, b.position.y + b.size.y / 2 });
        t.setPosition({ cx, y });
    };
    int score = 0;
    int lives = 3;
    float paddleW = PADDLE_W;
    float paddleX = 250;
    float ballX = paddleX + (paddleW - BALL_SIZE) / 2;
    float ballY = BALL_START_Y;
    float dx = 0, dy = 0;
    bool launched = false;

    int wideTimer = 0;

    int dxValues[4] = { -6, -3, 3, 6 };

    int selected = 0;
    std::string menuMessage = "";
    std::string menuItems[3] = { "New Game", "Continue", "Exit" };
    float time = 0;                                 // seconds since start
    float introTimer = 0;                           // bricks drop in
    float shake = 0;                                // screen shake strength
    float displayedScore = 0;                       // counts up towards score
    float paddleSquash = 0;                         // 1 right after a paddle hit
    float flash[ROWS][COLS] = {};                   // 1 right after a brick is hit
    std::vector<sf::Vector2f> trail;                // recent ball centres
    std::vector<Particle> particles;
    std::vector<Popup> popups;
    std::vector<Star> stars;
    for (int i = 0; i < 70; i++)
        stars.push_back({ frand(0, WINDOW_W), frand(0, WINDOW_H),
                          frand(10, 60), frand(1, 2.5f), frand(0, 6.28f) });

    // ---------- everything a new game needs ----------
    auto resetEffects = [&]()
    {
        introTimer = 0;
        shake = 0;
        paddleSquash = 0;
        trail.clear();
        powerActive = false;                        // power-up state resets too
        paddleW = PADDLE_W;
        wideTimer = 0;
        particles.clear();
        popups.clear();
        brickEvents.clear();
        for (int r = 0; r < ROWS; r++)
            for (int c = 0; c < COLS; c++)
                flash[r][c] = 0;
    };

    auto startNewGame = [&]()
    {
        loadLevel();
        score = 0;
        lives = 3;
        displayedScore = 0;
        paddleX = 250;
        ballX = paddleX + (paddleW - BALL_SIZE) / 2;
        ballY = BALL_START_Y;
        dx = 0;
        dy = 0;
        launched = false;
        menuMessage = "";
        resetEffects();
        state = PLAYING;
    };
    auto processBrickEvents = [&]()
    {
        const Theme& th = themes[themeIndex];
        for (const BrickEvent& e : brickEvents)
        {
            float cx = e.c * BRICK_W + BRICK_W / 2;
            float cy = BRICKS_Y + e.r * BRICK_H + BRICK_H / 2;
            sf::Color col = th.brick[e.before];

            flash[e.r][e.c] = 1.f;
            if (e.broke)
            {
                spawnBurst(particles, cx, cy, col, 18, 260);
                shake = std::max(shake, 5.f);
                popups.push_back({ cx, cy, 1.f, "+50", th.accent });

                spawnPowerUp(cx, cy);
            }
            else
            {
                spawnBurst(particles, cx, cy, col, 5, 120);
                popups.push_back({ cx, cy, 1.f, "+10", th.text });
            }
        }
        brickEvents.clear();
    };
    while (window.isOpen())
    {
        float dt = std::min(clock.restart().asSeconds(), 0.05f);
        time += dt;
        while (const std::optional event = window.pollEvent())
        {
            if (event->is<sf::Event::Closed>())
                window.close();

            if (const auto* key = event->getIf<sf::Event::KeyPressed>())
            {
                if (state == PLAYING)
                {
                    if (key->code == sf::Keyboard::Key::Escape)
                    {
                        saveGame(score, lives);
                        state = MENU;
                    }
                }
                else if (state == MENU)
                {
                    if (key->code == sf::Keyboard::Key::Up)
                        selected = (selected + 2) % 3;
                    if (key->code == sf::Keyboard::Key::Down)
                        selected = (selected + 1) % 3;
                    if (key->code == sf::Keyboard::Key::T)
                        themeIndex = (themeIndex + 1) % THEME_COUNT;
                    if (key->code == sf::Keyboard::Key::Enter)
                    {
                        if (selected == 0)
                            startNewGame();
                        else if (selected == 1)
                        {
                            if (loadGame(score, lives))
                            {
                                displayedScore = (float)score;
                                paddleX = 250;
                                ballX = paddleX + (paddleW - BALL_SIZE) / 2;
                                ballY = BALL_START_Y;
                                dx = 0;
                                dy = 0;
                                launched = false;
                                menuMessage = "";
                                resetEffects();
                                state = PLAYING;
                            }
                            else
                                menuMessage = "No saved game";
                        }
                        else
                            window.close();
                    }
                }
                else if (state == GAMEOVER || state == WIN)
                {
                    if (key->code == sf::Keyboard::Key::Enter)
                        state = MENU;
                }
            }
        }

        const Theme& th = themes[themeIndex];
        if (state == PLAYING)
        {
            if (wideTimer > 0)
            {
                wideTimer--;
                if (wideTimer == 0)
                    paddleW = PADDLE_W;
            }

            if (sf::Keyboard::isKeyPressed(sf::Keyboard::Key::Left) ||
                sf::Keyboard::isKeyPressed(sf::Keyboard::Key::A))
                paddleX -= PADDLE_SPEED;

            if (sf::Keyboard::isKeyPressed(sf::Keyboard::Key::Right) ||
                sf::Keyboard::isKeyPressed(sf::Keyboard::Key::D))
                paddleX += PADDLE_SPEED;

            if (paddleX < 0)
                paddleX = 0;
            if (paddleX > WINDOW_W - paddleW)
                paddleX = WINDOW_W - paddleW;

            if (!launched)
            {
                ballX = paddleX + (paddleW - BALL_SIZE) / 2;
                ballY = BALL_START_Y;

                if (sf::Keyboard::isKeyPressed(sf::Keyboard::Key::Space))
                {
                    launched = true;
                    dy = -5;
                    if (rand() % 2 == 0)
                        dx = -3;
                    else
                        dx = 3;
                    if (DEBUG) std::cout << "LAUNCH dx=" << dx << " dy=" << dy << "\n";
                }
            }
            else
            {
                ballX += dx;
                if (ballX < 0 || ballX + BALL_SIZE > WINDOW_W)
                {
                    ballX -= dx;
                    dx = -dx;
                    spawnBurst(particles, ballX < 300 ? 0.f : (float)WINDOW_W, ballY + 6, th.accent, 3, 90);
                    if (DEBUG) std::cout << "WALL bounce, dx=" << dx << "\n";
                }
                if (hitBricks(ballX, ballY, score))
                {
                    ballX -= dx;
                    dx = -dx;
                    processBrickEvents();
                }
                ballY += dy;
                if (ballY < HUD_H)
                {
                    ballY -= dy;
                    dy = -dy;
                    spawnBurst(particles, ballX + 6, (float)HUD_H, th.accent, 3, 90);
                    if (DEBUG) std::cout << "HUD bounce, dy=" << dy << "\n";
                }
                if (hitBricks(ballX, ballY, score))
                {
                    ballY -= dy;
                    dy = -dy;
                    processBrickEvents();
                }
                if (dy > 0 && ballX < paddleX + paddleW && ballX + BALL_SIZE > paddleX && ballY < PADDLE_Y + PADDLE_H && ballY + BALL_SIZE > PADDLE_Y)
                {
                    ballY = BALL_START_Y;
                    dy = -5;

                    int part = (int)((ballX + BALL_SIZE / 2 - paddleX) / (paddleW / 4));
                    if (part < 0) part = 0;
                    if (part > 3) part = 3;
                    dx = (float)dxValues[part];

                    paddleSquash = 1.f;
                    spawnBurst(particles, ballX + 6, PADDLE_Y, th.paddle, 6, 140);
                    if (DEBUG) std::cout << "PADDLE hit: part=" << part << " dx=" << dx << "\n";
                }
                if (ballY > WINDOW_H)
                {
                    lives--;
                    launched = false;
                    shake = 14.f;  
                    spawnBurst(particles, ballX + 6, (float)WINDOW_H - 10, th.accent, 20, 240);
                    if (DEBUG) std::cout << "BALL LOST, lives=" << lives << "\n";
                    if (lives == 0)
                    {
                        state = GAMEOVER;
                        clearSave();
                    }
                }
                if (state == PLAYING && countBricks() == 0)
                {
                    state = WIN;
                    clearSave();
                    spawnBurst(particles, 300.f, 300.f, th.accent, 60, 420);
                }
            }
            if (state == PLAYING)
            {
                char caught = updatePowerUp(paddleX, paddleW);

                if (caught == 'W')
                {
                    paddleW = WIDE_W;
                    wideTimer = WIDE_FRAMES;
                    popups.push_back({ paddleX + paddleW / 2, PADDLE_Y - 10.f, 1.f, "WIDE!", th.accent });
                }
                else if (caught == '+')
                {
                    lives++;
                    popups.push_back({ paddleX + paddleW / 2, PADDLE_Y - 10.f, 1.f, "+1 LIFE", th.accent });
                }

                if (caught != 0)
                    spawnBurst(particles, paddleX + paddleW / 2, PADDLE_Y, th.accent, 14, 200);
            }
            if (launched)
            {
                trail.push_back({ ballX + 6, ballY + 6 });
                if (trail.size() > 14)
                    trail.erase(trail.begin());
            }
            else
                trail.clear();

            introTimer += dt;
        }

        // ================= 2b. UPDATE: animations (always) =================
        for (Star& s : stars)
        {
            s.y += s.speed * dt;
            if (s.y > WINDOW_H) { s.y = 0; s.x = frand(0, WINDOW_W); }
        }

        for (Particle& p : particles)
        {
            p.vy += 400.f * dt;                    // gravity
            p.x += p.vx * dt;
            p.y += p.vy * dt;
            p.life -= dt;
        }
        particles.erase(std::remove_if(particles.begin(), particles.end(),
                        [](const Particle& p) { return p.life <= 0; }), particles.end());

        for (Popup& p : popups)
        {
            p.y -= 40.f * dt;
            p.life -= dt * 1.2f;
        }
        popups.erase(std::remove_if(popups.begin(), popups.end(),
                     [](const Popup& p) { return p.life <= 0; }), popups.end());

        for (int r = 0; r < ROWS; r++)
            for (int c = 0; c < COLS; c++)
                flash[r][c] = std::max(0.f, flash[r][c] - dt * 4.f);

        shake = std::max(0.f, shake - 40.f * dt);
        paddleSquash = std::max(0.f, paddleSquash - dt * 5.f);
        displayedScore += ((float)score - displayedScore) * std::min(1.f, dt * 8.f);

        if (state == WIN)                          // confetti rain
        {
            for (int i = 0; i < 2; i++)
            {
                sf::Color col = (rand() % 2) ? th.accent : th.brick[1 + rand() % 3];
                particles.push_back({ frand(0, WINDOW_W), -10.f, frand(-40, 40), frand(20, 80),
                                      3.f, 3.f, frand(4, 8), col });
            }
        }

        // ================= 3. DRAW =================
        window.clear(th.bg);

        sf::Vector2f off(frand(-1, 1) * shake, frand(-1, 1) * shake);
        view.setCenter({ WINDOW_W / 2.f + off.x, WINDOW_H / 2.f + off.y });
        window.setView(view);

        // ----- starfield -----
        for (const Star& s : stars)
        {
            float tw = 0.5f + 0.5f * std::sin(time * 2.f + s.phase);   // twinkle
            rectShape.setSize({ s.size, s.size });
            rectShape.setOrigin({ 0.f, 0.f });
            rectShape.setScale({ 1.f, 1.f });
            rectShape.setRotation(sf::degrees(0.f));
            rectShape.setPosition({ s.x, s.y });
            rectShape.setFillColor(withAlpha(th.star, 60 + 150 * tw));
            window.draw(rectShape);
        }

        if (state == MENU)
        {
            // ----- wavy title, one letter at a time -----
            std::string title = "BLOCK BUSTER";
            sf::Text full(font, title, 56);
            full.setStyle(sf::Text::Bold);
            sf::FloatRect fb = full.getLocalBounds();
            full.setPosition({ 300.f - fb.size.x / 2 - fb.position.x, 120.f });

            for (size_t i = 0; i < title.size(); i++)
            {
                sf::Vector2f pos = full.findCharacterPos(i);
                sf::Text letter(font, std::string(1, title[i]), 56);
                letter.setStyle(sf::Text::Bold);
                letter.setPosition({ pos.x, pos.y + std::sin(time * 3.f + i * 0.5f) * 10.f });
                float k = 0.5f + 0.5f * std::sin(time * 2.f + i * 0.6f);
                letter.setFillColor(mixColor(th.accent, th.brick[1], k));
                window.draw(letter);
            }

            // ----- row of little bricks under the title -----
            for (int i = 0; i < 10; i++)
            {
                float k = 0.5f + 0.5f * std::sin(time * 3.f - i * 0.6f);
                rectShape.setSize({ 52.f, 10.f });
                rectShape.setPosition({ i * 60.f + 4.f, 235.f });
                rectShape.setFillColor(withAlpha(th.brick[1 + i % 3], 90 + 165 * k));
                window.draw(rectShape);
            }

            // ----- menu items -----
            text.setStyle(sf::Text::Bold);
            for (int i = 0; i < 3; i++)
            {
                bool sel = (i == selected);
                float s = sel ? 1.f + 0.06f * std::sin(time * 6.f) : 1.f;
                placeText(text, sel ? "> " + menuItems[i] + " <" : menuItems[i], 32,
                          300.f, 330.f + i * 60.f);
                text.setScale({ s, s });
                text.setFillColor(sel ? th.accent : withAlpha(th.text, 200));
                window.draw(text);
            }
            text.setScale({ 1.f, 1.f });

            if (menuMessage != "")
            {
                placeText(text, menuMessage, 26, 300.f, 540.f);
                text.setFillColor(rgb(255, 80, 80));
                window.draw(text);
            }

            placeText(text, std::string("Theme: ") + th.name + "   (press T to change)",
                      18, 300.f, 620.f);
            text.setFillColor(withAlpha(th.text, 170));
            window.draw(text);
            placeText(text, "Up/Down: move     Enter: select", 16, 300.f, 650.f);
            text.setFillColor(withAlpha(th.text, 120));
            window.draw(text);
            text.setStyle(sf::Text::Regular);
        }
        else
        {
            // ----- HUD -----
            rectShape.setSize({ (float)WINDOW_W, (float)HUD_H });
            rectShape.setPosition({ 0.f, 0.f });
            rectShape.setFillColor(th.hud);
            window.draw(rectShape);

            rectShape.setSize({ (float)WINDOW_W, 2.f });
            rectShape.setPosition({ 0.f, (float)HUD_H - 2 });
            rectShape.setFillColor(th.accent);
            window.draw(rectShape);

            // ----- wide-paddle time left -----
            if (wideTimer > 0)
            {
                rectShape.setSize({ WINDOW_W * wideTimer / (float)WIDE_FRAMES, 3.f });
                rectShape.setPosition({ 0.f, (float)HUD_H });
                rectShape.setFillColor(withAlpha(th.paddle, 200));
                window.draw(rectShape);
            }

            // ----- bricks -----
            for (int r = 0; r < ROWS; r++)
            {
                for (int c = 0; c < COLS; c++)
                {
                    if (bricks[r][c] == 0) continue;

                    float delay = r * 0.07f + c * 0.025f;         // staggered drop-in
                    float p = std::clamp((introTimer - delay) / 0.45f, 0.f, 1.f);
                    if (p <= 0.f) continue;
                    float e = 1.f - (1.f - p) * (1.f - p) * (1.f - p);

                    float f = flash[r][c];
                    sf::Color base = th.brick[bricks[r][c]];
                    sf::Color col = mixColor(base, sf::Color::White, f * 0.8f);
                    col.a = (std::uint8_t)(255 * e);

                    float cx = c * BRICK_W + BRICK_W / 2;
                    float cy = BRICKS_Y + r * BRICK_H + BRICK_H / 2 - (1.f - e) * 140.f;
                    float sc = 1.f + 0.18f * f;                    // pop when hit

                    if (th.glow && p >= 1.f)
                    {
                        rectShape.setSize({ BRICK_W + 4, BRICK_H + 4 });
                        rectShape.setOrigin({ (BRICK_W + 4) / 2, (BRICK_H + 4) / 2 });
                        rectShape.setPosition({ cx, cy });
                        rectShape.setScale({ sc, sc });
                        rectShape.setFillColor(withAlpha(base, 40));
                        window.draw(rectShape, additive);
                    }

                    rectShape.setSize({ BRICK_W - 2, BRICK_H - 2 });
                    rectShape.setOrigin({ (BRICK_W - 2) / 2, (BRICK_H - 2) / 2 });
                    rectShape.setPosition({ cx, cy });
                    rectShape.setScale({ sc, sc });
                    rectShape.setFillColor(col);
                    window.draw(rectShape);

                    rectShape.setSize({ BRICK_W - 8, 3.f });       // shine strip
                    rectShape.setOrigin({ (BRICK_W - 8) / 2, 0.f });
                    rectShape.setPosition({ cx, cy - (BRICK_H - 2) / 2 * sc + 2.f });
                    rectShape.setFillColor(withAlpha(sf::Color::White, 70 * e));
                    window.draw(rectShape);
                }
            }
            rectShape.setOrigin({ 0.f, 0.f });
            rectShape.setScale({ 1.f, 1.f });

            // ----- paddle (copy our variables into the shape) -----
            float px = paddleX + paddleW / 2;
            float py = PADDLE_Y + PADDLE_H / 2;
            float sy = 1.f + 0.7f * paddleSquash;
            float sx = 1.f + 0.06f * paddleSquash;

            if (th.glow)
            {
                rectShape.setSize({ paddleW + 20, PADDLE_H + 14 });
                rectShape.setOrigin({ (paddleW + 20) / 2, (PADDLE_H + 14) / 2 });
                rectShape.setPosition({ px, py });
                rectShape.setFillColor(withAlpha(th.paddle, 45));
                window.draw(rectShape, additive);
            }
            rectShape.setSize({ paddleW, PADDLE_H });
            rectShape.setOrigin({ paddleW / 2, PADDLE_H / 2 });
            rectShape.setPosition({ px, py });
            rectShape.setScale({ sx, sy });
            rectShape.setFillColor(mixColor(th.paddle, sf::Color::White, paddleSquash * 0.7f));
            window.draw(rectShape);
            rectShape.setOrigin({ 0.f, 0.f });
            rectShape.setScale({ 1.f, 1.f });

            // ----- ball trail, glow and ball -----
            for (size_t i = 0; i < trail.size(); i++)
            {
                float k = (i + 1) / (float)trail.size();
                float rad = 6.f * k;
                circleShape.setRadius(rad);
                circleShape.setOrigin({ rad, rad });
                circleShape.setPosition(trail[i]);
                circleShape.setFillColor(withAlpha(th.ball, 120 * k));
                window.draw(circleShape);
            }

            float bx = ballX + 6, by = ballY + 6;
            if (th.glow)
            {
                float radii[2] = { 18.f, 10.f };
                float alphas[2] = { 35.f, 70.f };
                for (int i = 0; i < 2; i++)
                {
                    circleShape.setRadius(radii[i]);
                    circleShape.setOrigin({ radii[i], radii[i] });
                    circleShape.setPosition({ bx, by });
                    circleShape.setFillColor(withAlpha(th.ball, alphas[i]));
                    window.draw(circleShape, additive);
                }
            }
            circleShape.setRadius(BALL_SIZE / 2);
            circleShape.setOrigin({ BALL_SIZE / 2, BALL_SIZE / 2 });
            circleShape.setPosition({ bx, by });
            circleShape.setFillColor(th.ball);
            window.draw(circleShape);

            // ----- falling power-up -----
            if (powerActive)
            {
                sf::Color pc = (powerType == 'W') ? th.accent : rgb(80, 220, 120);
                float pcx = powerX + POWERUP_SIZE / 2;
                float pcy = powerY + POWERUP_SIZE / 2;
                float pulse = 1.f + 0.1f * std::sin(time * 10.f);

                if (th.glow)
                {
                    rectShape.setSize({ POWERUP_SIZE + 14, POWERUP_SIZE + 14 });
                    rectShape.setOrigin({ (POWERUP_SIZE + 14) / 2, (POWERUP_SIZE + 14) / 2 });
                    rectShape.setPosition({ pcx, pcy });
                    rectShape.setScale({ pulse, pulse });
                    rectShape.setFillColor(withAlpha(pc, 60));
                    window.draw(rectShape, additive);
                }
                rectShape.setSize({ POWERUP_SIZE, POWERUP_SIZE });
                rectShape.setOrigin({ POWERUP_SIZE / 2, POWERUP_SIZE / 2 });
                rectShape.setPosition({ pcx, pcy });
                rectShape.setScale({ pulse, pulse });
                rectShape.setFillColor(pc);
                window.draw(rectShape);
                rectShape.setOrigin({ 0.f, 0.f });
                rectShape.setScale({ 1.f, 1.f });

                text.setStyle(sf::Text::Bold);
                placeText(text, std::string(1, powerType), 18, pcx, pcy);
                text.setFillColor(th.bg);
                window.draw(text);
            }

            // ----- particles -----
            for (const Particle& p : particles)
            {
                float k = p.life / p.maxLife;
                rectShape.setSize({ p.size * k + 1.f, p.size * k + 1.f });
                rectShape.setOrigin({ (p.size * k + 1.f) / 2, (p.size * k + 1.f) / 2 });
                rectShape.setRotation(sf::degrees(p.life * 360.f));
                rectShape.setPosition({ p.x, p.y });
                rectShape.setFillColor(withAlpha(p.color, 255 * k));
                if (th.glow) window.draw(rectShape, additive);
                else         window.draw(rectShape);
            }
            rectShape.setOrigin({ 0.f, 0.f });
            rectShape.setRotation(sf::degrees(0.f));

            // ----- score popups -----
            text.setStyle(sf::Text::Bold);
            for (const Popup& p : popups)
            {
                placeText(text, p.text, 20, p.x, p.y);
                text.setFillColor(withAlpha(p.color, 255 * p.life));
                window.draw(text);
            }

            // ----- HUD text -----
            placeText(text, "SCORE  " + std::to_string((int)(displayedScore + 0.5f)), 22, 90.f, 24.f);
            text.setFillColor(th.text);
            window.draw(text);

            for (int i = 0; i < lives; i++)                       // lives = glowing circles
            {
                float pulse = (lives == 1) ? 1.f + 0.25f * std::sin(time * 8.f) : 1.f;
                float rad = 7.f * pulse;
                circleShape.setRadius(rad);
                circleShape.setOrigin({ rad, rad });
                circleShape.setPosition({ WINDOW_W - 30.f - i * 24.f, 25.f });
                circleShape.setFillColor(th.accent);
                window.draw(circleShape);
            }

            if (state == PLAYING && !launched)
            {
                float blink = 0.5f + 0.5f * std::sin(time * 4.f);
                placeText(text, "Press SPACE to launch", 20, 300.f, 580.f);
                text.setFillColor(withAlpha(th.text, 90 + 150 * blink));
                window.draw(text);
            }
            text.setStyle(sf::Text::Regular);

            // ----- scanlines (glow themes only) -----
            if (th.glow)
            {
                rectShape.setSize({ (float)WINDOW_W, 1.f });
                rectShape.setFillColor(sf::Color(0, 0, 0, 40));
                for (int y = 0; y < WINDOW_H; y += 4)
                {
                    rectShape.setPosition({ 0.f, (float)y });
                    window.draw(rectShape);
                }
            }

            // ----- end screens -----
            if (state == GAMEOVER || state == WIN)
            {
                rectShape.setSize({ (float)WINDOW_W, (float)WINDOW_H });
                rectShape.setPosition({ 0.f, 0.f });
                rectShape.setFillColor(withAlpha(th.bg, 170));
                window.draw(rectShape);

                // confetti and sparks stay in front of the dim layer
                for (const Particle& p : particles)
                {
                    float k = p.life / p.maxLife;
                    rectShape.setSize({ p.size, p.size });
                    rectShape.setOrigin({ p.size / 2, p.size / 2 });
                    rectShape.setRotation(sf::degrees(p.life * 360.f));
                    rectShape.setPosition({ p.x, p.y });
                    rectShape.setFillColor(withAlpha(p.color, 255 * k));
                    window.draw(rectShape);
                }
                rectShape.setOrigin({ 0.f, 0.f });
                rectShape.setRotation(sf::degrees(0.f));

                text.setStyle(sf::Text::Bold);
                float s = 1.f + 0.05f * std::sin(time * 4.f);
                placeText(text, state == WIN ? "YOU WIN" : "GAME OVER", 56, 300.f, 270.f);
                text.setScale({ s, s });
                text.setFillColor(state == WIN ? th.accent : rgb(255, 80, 80));
                window.draw(text);
                text.setScale({ 1.f, 1.f });

                placeText(text, "Final score: " + std::to_string(score), 28, 300.f, 360.f);
                text.setFillColor(th.text);
                window.draw(text);

                float blink = 0.5f + 0.5f * std::sin(time * 4.f);
                placeText(text, "Press Enter", 24, 300.f, 410.f);
                text.setFillColor(withAlpha(th.text, 90 + 160 * blink));
                window.draw(text);
                text.setStyle(sf::Text::Regular);
            }
        }

        window.display();
    }

    return 0;
}
