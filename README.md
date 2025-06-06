import pygame
import random
import sys

# Initialisierung
pygame.init()
screen_width = 800
screen_height = 400
screen = pygame.display.set_mode((screen_width, screen_height))
pygame.display.set_caption("Jump & Slide Game")
clock = pygame.time.Clock()

# Farben
WHITE = (255, 255, 255)
BLACK = (0, 0, 0)
RED = (255, 0, 0)
GREEN = (0, 128, 0)
BLUE = (0, 0, 255)

# Spieler
class Player:
    def __init__(self):
        self.width = 40
        self.original_height = 60
        self.height = self.original_height
        self.x = 100
        self.y = screen_height - self.height - 50
        self.speed = 5
        self.jumping = False
        self.jump_velocity = 16  # Höherer Sprung
        self.jump_count = self.jump_velocity
        self.gravity = 0.8
        self.sliding = False
        self.slide_height = 30
        self.slide_cooldown = 0

    def update(self):
        keys = pygame.key.get_pressed()
        
        # Bewegung
        if keys[pygame.K_LEFT]:
            self.x -= self.speed
        if keys[pygame.K_RIGHT]:
            self.x += self.speed

        # Sprung (höher)
        if not self.jumping:
            if keys[pygame.K_UP] or keys[pygame.K_SPACE]:
                self.jumping = True
        else:
            if self.jump_count >= -self.jump_velocity:
                neg = 1
                if self.jump_count < 0:
                    neg = -1
                self.y -= (self.jump_count ** 2) * 0.5 * neg
                self.jump_count -= 1
            else:
                self.jumping = False
                self.jump_count = self.jump_velocity

        # Slide (unter Hindernisse)
        if (keys[pygame.K_DOWN] or keys[pygame.K_LSHIFT]) and not self.jumping and self.slide_cooldown == 0:
            self.sliding = True
            self.height = self.slide_height
        else:
            if not (keys[pygame.K_DOWN] or keys[pygame.K_LSHIFT]):
                self.sliding = False
                self.height = self.original_height

        if self.slide_cooldown > 0:
            self.slide_cooldown -= 1

        # Gravitation
        if self.y < screen_height - self.height - 50 and not self.jumping:
            self.y += self.gravity
        else:
            self.y = screen_height - self.height - 50

        # Bildschirmbegrenzung
        self.x = max(0, min(self.x, screen_width - self.width))

    def draw(self):
        pygame.draw.rect(screen, RED, (self.x, self.y, self.width, self.height))

    def get_rect(self):
        return pygame.Rect(self.x, self.y, self.width, self.height)

# Hindernisse
class Obstacle:
    def __init__(self, x):
        self.width = 50
        self.height = random.choice([60, 80, 100])  # Verschiedene Höhen
        self.x = x
        self.y = screen_height - self.height - 50
        self.passed = False

    def update(self, speed):
        self.x -= speed

    def draw(self):
        pygame.draw.rect(screen, GREEN, (self.x, self.y, self.width, self.height))

    def get_rect(self):
        return pygame.Rect(self.x, self.y, self.width, self.height)

# Spiel-Logik
class Game:
    def __init__(self):
        self.player = Player()
        self.obstacles = []
        self.score = 0
        self.game_speed = 5
        self.obstacle_timer = 0
        self.obstacle_frequency = 1500  # ms
        self.font = pygame.font.SysFont(None, 36)
        self.game_over = False

    def spawn_obstacle(self):
        now = pygame.time.get_ticks()
        if now - self.obstacle_timer > self.obstacle_frequency:
            self.obstacle_timer = now
            self.obstacles.append(Obstacle(screen_width))

    def update(self):
        if self.game_over:
            return

        self.player.update()
        self.spawn_obstacle()

        for obstacle in self.obstacles[:]:
            obstacle.update(self.game_speed)

            # Kollisionsprüfung mit Slide-Logik
            if self.player.get_rect().colliderect(obstacle.get_rect()):
                if self.player.sliding and (self.player.y + self.player.height) >= (obstacle.y + obstacle.height - 5):
                    # Erfolgreich unter dem Hindernis durchgerutscht
                    pass
                else:
                    self.game_over = True

            # Punkte zählen
            if obstacle.x + obstacle.width < self.player.x and not obstacle.passed:
                obstacle.passed = True
                self.score += 1

            # Hindernis entfernen
            if obstacle.x < -obstacle.width:
                self.obstacles.remove(obstacle)

        # Schwierigkeit erhöhen
        self.game_speed = 5 + self.score // 5

    def draw(self):
        screen.fill(WHITE)
        # Boden
        pygame.draw.rect(screen, BLACK, (0, screen_height - 50, screen_width, 50))
        
        # Spieler und Hindernisse
        self.player.draw()
        for obstacle in self.obstacles:
            obstacle.draw()
        
        # Score
        score_text = self.font.render(f"Score: {int(self.score)}", True, BLUE)
        screen.blit(score_text, (10, 10))
        
        if self.game_over:
            over_text = self.font.render("Game Over! Drücke R zum Neustart", True, BLACK)
            screen.blit(over_text, (screen_width//2 - 180, screen_height//2 - 18))

    def reset(self):
        self.player = Player()
        self.obstacles = []
        self.score = 0
        self.game_speed = 5
        self.game_over = False

# Hauptspiel-Schleife
def main():
    game = Game()
    running = True
    
    while running:
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                running = False
            if event.type == pygame.KEYDOWN:
                if event.key == pygame.K_r and game.game_over:
                    game.reset()
        
        game.update()
        game.draw()
        
        pygame.display.flip()
        clock.tick(60)

    pygame.quit()
    sys.exit()

if __name__ == "__main__":
    main()# TeST
