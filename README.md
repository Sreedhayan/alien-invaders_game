// Supercharged Space Invaders - FULL CODE

// Player
float playerX, playerY, playerSpeed = 6;
int playerHealth = 3;
int shieldStrength = 100;
boolean shieldActive = true;
boolean rapidFire = false;
int rapidFireTimer = 0;
int fireRate = 15;
int fireCounter = 0;

// Bullets
ArrayList<Bullet> playerBullets;
ArrayList<EnemyBullet> enemyBullets;

// Enemies
ArrayList<Enemy> enemies;
int rows = 5;
int cols = 9;
float enemySpacingX = 55;
float enemySpacingY = 50;
float enemyStartX = 100;
float enemyStartY = 60;
float enemySpeedX = 1.5;
boolean moveRight = true;
float enemyDescend = 20;

// Boss & MiniBoss
Boss boss;
MiniBoss miniBoss;
boolean bossActive = false;
boolean miniBossActive = false;

// Levels
int level = 1;

// Score
int score = 0;
int comboCounter = 0;

// Game state
boolean gameOver = false;
boolean paused = false;
boolean titleScreen = true;
boolean fadeTransition = false;
float fadeAlpha = 255;

// Particles
ArrayList<Particle> particles;

// Power-ups
ArrayList<PowerUp> powerUps;

// Font
PFont font;

void setup() {
  size(800, 600);
  font = createFont("Arial Bold", 24);
  restartGame();
}

void draw() {
  background(10, 10, 30);
  drawStarfield();
  
  if (titleScreen) {
    drawTitleScreen();
  } else if (fadeTransition) {
    fadeAlpha -= 5;
    if (fadeAlpha <= 0) {
      fadeAlpha = 0;
      fadeTransition = false;
    }
    fill(0, fadeAlpha);
    rect(0, 0, width, height);
  } else if (!gameOver && !paused) {
    drawHUD();
    drawPlayer();
    handleBullets();
    
    if (bossActive) handleBoss();
    else if (miniBossActive) handleMiniBoss();
    else handleEnemies();
    
    handleParticles();
    handlePowerUps();
    
    fireCounter++;
    
    if (rapidFire) {
      rapidFireTimer--;
      if (rapidFireTimer <= 0) {
        rapidFire = false;
        fireRate = 15;
      }
    }
  } else if (paused) {
    fill(255);
    textFont(font);
    textSize(36);
    textAlign(CENTER, CENTER);
    text("PAUSED", width / 2, height / 2);
  } else {
    fill(255, 0, 0);
    textFont(font);
    textSize(48);
    textAlign(CENTER, CENTER);
    text("GAME OVER", width / 2, height / 2);
    fill(255);
    textSize(24);
    text("Press 'R' to Restart", width / 2, height / 2 + 50);
  }
}

void drawTitleScreen() {
  fill(255);
  textFont(font);
  textSize(48);
  textAlign(CENTER, CENTER);
  text("SUPER SPACE INVADERS", width / 2, height / 2 - 50);
  textSize(24);
  text("Press ENTER to Start", width / 2, height / 2 + 20);
}

void drawHUD() {
  fill(255);
  textFont(font);
  text("Score: " + score, 10, 30);
  text("Level: " + level, width - 150, 30);
  text("Combo: " + comboCounter, width / 2 - 50, 30);
  
  fill(255, 0, 0);
  rect(10, 50, playerHealth * 40, 20);
  fill(255);
  text("Health", 10, 45);
  
  fill(0, 0, 255, 150);
  rect(10, 80, shieldStrength, 15);
  fill(255);
  text("Shield", 10, 75);
  
  if (rapidFire) {
    fill(255, 255, 0);
    text("Rapid Fire!", width / 2 - 50, 60);
  }
}

void drawPlayer() {
  pushMatrix();
  translate(playerX, playerY);
  noStroke();
  fill(0, 255, 0);
  triangle(-20, 20, 20, 20, 0, -20);
  popMatrix();
  
  if (keyPressed) {
    if (keyCode == LEFT) playerX -= playerSpeed;
    else if (keyCode == RIGHT) playerX += playerSpeed;
  }
  playerX = constrain(playerX, 20, width - 20);
}

void handleBullets() {
  for (int i = playerBullets.size() - 1; i >= 0; i--) {
    Bullet b = playerBullets.get(i);
    b.update();
    b.display();
    if (b.offScreen()) playerBullets.remove(i);
    else {
      if (!bossActive && !miniBossActive) {
        for (int j = enemies.size() - 1; j >= 0; j--) {
          Enemy e = enemies.get(j);
          if (e.hits(b)) {
            explode(e.x, e.y);
            enemies.remove(j);
            playerBullets.remove(i);
            score += 10 * (comboCounter + 1);
            comboCounter++;
            if (random(1) < 0.1) powerUps.add(new PowerUp(e.x, e.y));
            break;
          }
        }
      } else if (bossActive && boss.hits(b)) {
        explode(boss.x, boss.y);
        boss.health -= 5;
        playerBullets.remove(i);
        if (boss.health <= 0) {
          bossActive = false;
          score += 200;
          level++;
          nextLevel();
        }
      } else if (miniBossActive && miniBoss.hits(b)) {
        explode(miniBoss.x, miniBoss.y);
        miniBoss.health -= 5;
        playerBullets.remove(i);
        if (miniBoss.health <= 0) {
          miniBossActive = false;
          score += 100;
          level++;
          nextLevel();
        }
      }
    }
  }
  
  for (int i = enemyBullets.size() - 1; i >= 0; i--) {
    EnemyBullet eb = enemyBullets.get(i);
    eb.update();
    eb.display();
    if (eb.offScreen()) enemyBullets.remove(i);
    else if (eb.hits(playerX, playerY)) {
      if (shieldActive && shieldStrength > 0) {
        shieldStrength -= 25;
        if (shieldStrength <= 0) shieldActive = false;
      } else {
        playerHealth--;
        comboCounter = 0;
        screenShake();
        if (playerHealth <= 0) gameOver = true;
      }
      enemyBullets.remove(i);
    }
  }
}

void handleEnemies() {
  boolean reachedEdge = false;
  for (Enemy e : enemies) {
    e.update(enemySpeedX, moveRight, level);
    if ((e.x > width - 40 && moveRight) || (e.x < 40 && !moveRight)) reachedEdge = true;
  }
  if (reachedEdge) {
    moveRight = !moveRight;
    for (Enemy e : enemies) {
      e.y += enemyDescend;
      if (e.y >= playerY - 40) gameOver = true;
    }
  }
  for (Enemy e : enemies) {
    e.display();
    if (random(1) < 0.001 * level) enemyBullets.add(new EnemyBullet(e.x, e.y));
  }
  if (enemies.isEmpty()) {
    if (level % 5 == 0) {
      boss = new Boss(width / 2, 100, 200 + level * 20);
      bossActive = true;
    } else if (level % 3 == 0) {
      miniBoss = new MiniBoss(random(100, width - 100), 100, 100 + level * 10);
      miniBossActive = true;
    } else {
      level++;
      nextLevel();
    }
  }
}

void handleBoss() {
  boss.update();
  boss.display();
  if (random(1) < 0.02) enemyBullets.add(new EnemyBullet(boss.x, boss.y + 30));
}

void handleMiniBoss() {
  miniBoss.update();
  miniBoss.display();
  if (random(1) < 0.03) enemyBullets.add(new EnemyBullet(miniBoss.x, miniBoss.y + 20));
}

void handleParticles() {
  for (int i = particles.size() - 1; i >= 0; i--) {
    Particle p = particles.get(i);
    p.update();
    p.display();
    if (p.isDead()) particles.remove(i);
  }
}

void handlePowerUps() {
  for (int i = powerUps.size() - 1; i >= 0; i--) {
    PowerUp pu = powerUps.get(i);
    pu.update();
    pu.display();
    if (pu.hits(playerX, playerY)) {
      activatePowerUp(pu.type);
      powerUps.remove(i);
    }
  }
}

void activatePowerUp(String type) {
  if (type.equals("Shield")) {
    shieldStrength = 100;
    shieldActive = true;
  } else if (type.equals("RapidFire")) {
    rapidFire = true;
    rapidFireTimer = 600; // lasts ~10 seconds
    fireRate = 4;
  } else if (type.equals("Life")) {
    playerHealth++;
  }
}

void screenShake() {
  translate(random(-5, 5), random(-5, 5));
}

void explode(float x, float y) {
  for (int i = 0; i < 20; i++) particles.add(new Particle(x, y));
}

void drawStarfield() {
  for (int i = 0; i < 200; i++) {
    float layer = random(1, 3);
    stroke(255, 100 / layer);
    point(random(width), random(height));
  }
}

void keyPressed() {
  if (titleScreen && keyCode == ENTER) {
    titleScreen = false;
    fadeAlpha = 255;
    fadeTransition = true;
  }
  if (!gameOver && !titleScreen) {
    if (key == ' ' || key == 'z' || key == 'Z') {
      if (fireCounter >= fireRate || rapidFire) {
        playerBullets.add(new Bullet(playerX, playerY - 20));
        fireCounter = 0;
      }
    } else if (key == 'p' || key == 'P') paused = !paused;
  } else if (gameOver && (key == 'r' || key == 'R')) {
    restartGame();
  }
}

void restartGame() {
  playerX = width / 2;
  playerY = height - 60;
  playerBullets = new ArrayList<Bullet>();
  enemyBullets = new ArrayList<EnemyBullet>();
  enemies = new ArrayList<Enemy>();
  particles = new ArrayList<Particle>();
  powerUps = new ArrayList<PowerUp>();
  moveRight = true;
  enemySpeedX = 1.5;
  playerHealth = 3;
  shieldStrength = 100;
  shieldActive = true;
  rapidFire = false;
  fireRate = 15;
  fireCounter = 0;
  level = 1;
  score = 0;
  comboCounter = 0;
  bossActive = false;
  miniBossActive = false;
  gameOver = false;
  paused = false;
  titleScreen = true;
  fadeTransition = false;
  fadeAlpha = 255;
  setupEnemies();
}

void setupEnemies() {
  enemies.clear();
  for (int r = 0; r < rows; r++) {
    for (int c = 0; c < cols; c++) {
      float x = enemyStartX + c * enemySpacingX;
      float y = enemyStartY + r * enemySpacingY;
      enemies.add(new Enemy(x, y));
    }
  }
}

void nextLevel() {
  enemySpeedX += 0.2;
  setupEnemies();
  shieldActive = true;
  shieldStrength = 100;
  comboCounter = 0;
}

// ======= Classes =======

class Bullet {
  float x, y;
  float speed = 8;
  
  Bullet(float x, float y) {
    this.x = x;
    this.y = y;
  }
  
  void update() {
    y -= speed;
  }
  
  void display() {
    stroke(255, 255, 0);
    strokeWeight(4);
    line(x, y, x, y - 10);
  }
  
  boolean offScreen() {
    return y < 0;
  }
}

class EnemyBullet {
  float x, y;
  float speed = 5;
  
  EnemyBullet(float x, float y) {
    this.x = x;
    this.y = y;
  }
  
  void update() {
    y += speed;
  }
  
  void display() {
    stroke(255, 0, 0);
    strokeWeight(3);
    line(x, y, x, y + 10);
  }
  
  boolean offScreen() {
    return y > height;
  }
  
  boolean hits(float px, float py) {
    return dist(x, y, px, py) < 20;
  }
}

class Enemy {
  float x, y;
  float size = 30;
  
  Enemy(float x, float y) {
    this.x = x;
    this.y = y;
  }
  
  void update(float speedX, boolean moveRight, int level) {
    if (moveRight) x += speedX;
    else x -= speedX;
  }
  
  void display() {
    fill(255, 0, 0);
    rectMode(CENTER);
    rect(x, y, size, size);
    // add some cool design
    fill(0);
    rect(x, y, size * 0.6, size * 0.6);
    stroke(255, 100, 100);
    strokeWeight(2);
    line(x - size/3, y - size/3, x + size/3, y + size/3);
    line(x + size/3, y - size/3, x - size/3, y + size/3);
    noStroke();
  }
  
  boolean hits(Bullet b) {
    return dist(x, y, b.x, b.y) < size / 2;
  }
}

class Boss {
  float x, y;
  float w = 80;
  float h = 40;
  int health;
  
  Boss(float x, float y, int health) {
    this.x = x;
    this.y = y;
    this.health = health;
  }
  
  void update() {
    x += sin(frameCount * 0.05) * 2;
  }
  
  void display() {
    fill(255, 0, 255);
    rectMode(CENTER);
    rect(x, y, w, h);
    fill(255);
    textAlign(CENTER);
    text("Boss HP: " + health, x, y - 30);
    // Boss eyes
    fill(0);
    ellipse(x - 20, y, 15, 20);
    ellipse(x + 20, y, 15, 20);
    fill(255, 0, 0);
    ellipse(x - 20, y, 7, 10);
    ellipse(x + 20, y, 7, 10);
  }
  
  boolean hits(Bullet b) {
    return b.x > x - w/2 && b.x < x + w/2 && b.y > y - h/2 && b.y < y + h/2;
  }
}

class MiniBoss {
  float x, y;
  float size = 60;
  int health;
  
  MiniBoss(float x, float y, int health) {
    this.x = x;
    this.y = y;
    this.health = health;
  }
  
  void update() {
    x += sin(frameCount * 0.1) * 3;
  }
  
  void display() {
    fill(0, 255, 255);
    ellipse(x, y, size, size);
    fill(255);
    textAlign(CENTER);
    text("MiniBoss HP: " + health, x, y - 40);
    // MiniBoss face
    fill(0);
    ellipse(x - 15, y, 10, 15);
    ellipse(x + 15, y, 10, 15);
    fill(255, 0, 0);
    ellipse(x - 15, y, 5, 10);
    ellipse(x + 15, y, 5, 10);
  }
  
  boolean hits(Bullet b) {
    return dist(x, y, b.x, b.y) < size / 2;
  }
}

class Particle {
  float x, y;
  float vx, vy;
  float alpha = 255;
  
  Particle(float x, float y) {
    this.x = x;
    this.y = y;
    vx = random(-2, 2);
    vy = random(-2, 2);
  }
  
  void update() {
    x += vx;
    y += vy;
    alpha -= 5;
  }
  
  void display() {
    noStroke();
    fill(255, alpha);
    ellipse(x, y, 5, 5);
  }
  
  boolean isDead() {
    return alpha <= 0;
  }
}

class PowerUp {
  float x, y;
  String type;
  float speed = 3;
  
  PowerUp(float x, float y) {
    this.x = x;
    this.y = y;
    float r = random(1);
    if (r < 0.33) type = "Shield";
    else if (r < 0.66) type = "RapidFire";
    else type = "Life";
  }
  
  void update() {
    y += speed;
  }
  
  void display() {
    fill(255);
    rectMode(CENTER);
    if (type.equals("Shield")) fill(0, 0, 255);
    else if (type.equals("RapidFire")) fill(255, 255, 0);
    else fill(0, 255, 0);
    rect(x, y, 20, 20);
    fill(0);
    textAlign(CENTER, CENTER);
    textSize(12);
    text(type.charAt(0), x, y);
  }
  
  boolean hits(float px, float py) {
    return dist(x, y, px, py) < 20;
  }
}
