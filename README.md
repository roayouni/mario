import pygame
import sys
import random
import math
import json
from enum import Enum
from dataclasses import dataclass
from typing import List, Tuple, Optional

# Initialize Pygame
pygame.init()
pygame.mixer.init()

# Constants
SCREEN_WIDTH = 1200
SCREEN_HEIGHT = 700
FPS = 60
GRAVITY = 0.8
MAX_FALL_SPEED = 15

# Colors
COLORS = {
    'WHITE': (255, 255, 255),
    'BLACK': (0, 0, 0),
    'RED': (255, 0, 0),
    'GREEN': (0, 255, 0),
    'BLUE': (0, 0, 255),
    'YELLOW': (255, 255, 0),
    'BROWN': (139, 69, 19),
    'GRAY': (128, 128, 128),
    'ORANGE': (255, 165, 0),
    'DARK_BROWN': (101, 67, 33),
    'LIGHT_BLUE': (173, 216, 230),
    'DARK_GREEN': (0, 128, 0),
    'PURPLE': (128, 0, 128),
    'GOLD': (255, 215, 0),
    'SILVER': (192, 192, 192)
}

class GameState(Enum):
    MENU = 1
    PLAYING = 2
    PAUSED = 3
    GAME_OVER = 4
    LEVEL_COMPLETE = 5
    VICTORY = 6

@dataclass
class GameConfig:
    player_speed: float = 6.0
    jump_power: float = 16.0
    enemy_speed: float = 2.0
    coin_value: int = 100
    level_bonus: int = 1000
    max_lives: int = 3

class SoundManager:
    def __init__(self):
        self.sounds = {}
        self.music_volume = 0.7
        self.sfx_volume = 0.8
        self.load_sounds()
    
    def load_sounds(self):
        # Create simple sound effects using pygame's sound generation
        try:
            # Jump sound (simple beep)
            self.create_jump_sound()
            # Coin sound
            self.create_coin_sound()
            # Game over sound
            self.create_game_over_sound()
        except:
            print("Warning: Could not create sound effects")
    
    def create_jump_sound(self):
        # Create a simple jump sound effect
        duration = 0.1
        sample_rate = 22050
        frames = int(duration * sample_rate)
        arr = []
        for i in range(frames):
            wave = 4096 * math.sin(2 * math.pi * 440 * i / sample_rate)
            wave = int(wave * (1 - i / frames))  # Fade out
            arr.append([wave, wave])
        sound = pygame.sndarray.make_sound(pygame.array.array('h', arr))
        self.sounds['jump'] = sound
    
    def create_coin_sound(self):
        duration = 0.2
        sample_rate = 22050
        frames = int(duration * sample_rate)
        arr = []
        for i in range(frames):
            wave = 2048 * math.sin(2 * math.pi * 880 * i / sample_rate)
            wave = int(wave * (1 - i / frames))
            arr.append([wave, wave])
        sound = pygame.sndarray.make_sound(pygame.array.array('h', arr))
        self.sounds['coin'] = sound
    
    def create_game_over_sound(self):
        duration = 1.0
        sample_rate = 22050
        frames = int(duration * sample_rate)
        arr = []
        for i in range(frames):
            wave = 3072 * math.sin(2 * math.pi * 220 * i / sample_rate)
            wave = int(wave * (1 - i / frames))
            arr.append([wave, wave])
        sound = pygame.sndarray.make_sound(pygame.array.array('h', arr))
        self.sounds['game_over'] = sound
    
    def play_sound(self, name: str):
        if name in self.sounds:
            sound = self.sounds[name]
            sound.set_volume(self.sfx_volume)
            sound.play()

class ParticleSystem:
    def __init__(self):
        self.particles = []
    
    def add_coin_effect(self, x: int, y: int):
        for _ in range(8):
            particle = {
                'x': x,
                'y': y,
                'vel_x': random.uniform(-3, 3),
                'vel_y': random.uniform(-5, -1),
                'life': 30,
                'color': COLORS['GOLD']
            }
            self.particles.append(particle)
    
    def add_jump_effect(self, x: int, y: int):
        for _ in range(5):
            particle = {
                'x': x,
                'y': y,
                'vel_x': random.uniform(-2, 2),
                'vel_y': random.uniform(0, 2),
                'life': 20,
                'color': COLORS['WHITE']
            }
            self.particles.append(particle)
    
    def update(self):
        for particle in self.particles[:]:
            particle['x'] += particle['vel_x']
            particle['y'] += particle['vel_y']
            particle['vel_y'] += 0.2  # Gravity
            particle['life'] -= 1
            
            if particle['life'] <= 0:
                self.particles.remove(particle)
    
    def draw(self, screen):
        for particle in self.particles:
            alpha = max(0, particle['life'] * 8)
            size = max(1, particle['life'] // 5)
            pygame.draw.circle(screen, particle['color'], 
                             (int(particle['x']), int(particle['y'])), size)

class Animation:
    def __init__(self, frames: List[pygame.Surface], frame_duration: int):
        self.frames = frames
        self.frame_duration = frame_duration
        self.current_frame = 0
        self.frame_timer = 0
    
    def update(self):
        self.frame_timer += 1
        if self.frame_timer >= self.frame_duration:
            self.frame_timer = 0
            self.current_frame = (self.current_frame + 1) % len(self.frames)
    
    def get_current_frame(self) -> pygame.Surface:
        return self.frames[self.current_frame]

class Player:
    def __init__(self, x: int, y: int, config: GameConfig):
        self.rect = pygame.Rect(x, y, 32, 42)
        self.vel_x = 0.0
        self.vel_y = 0.0
        self.config = config
        self.on_ground = False
        self.lives = config.max_lives
        self.score = 0
        self.facing_right = True
        self.animation_timer = 0
        self.invulnerable = False
        self.invulnerable_timer = 0
        self.spawn_x = x
        self.spawn_y = y
        
        # Create player sprites
        self.create_sprites()
        
    def create_sprites(self):
        # Create different sprites for different states
        self.sprites = {}
        
        # Standing sprite
        standing = pygame.Surface((32, 42), pygame.SRCALPHA)
        # Mario's body
        pygame.draw.rect(standing, COLORS['BLUE'], (8, 18, 16, 20))  # Overalls
        pygame.draw.rect(standing, COLORS['RED'], (6, 8, 20, 12))   # Shirt
        # Mario's head
        pygame.draw.circle(standing, (255, 220, 177), (16, 12), 8)  # Head
        # Hat
        pygame.draw.circle(standing, COLORS['RED'], (16, 8), 10)
        # Eyes
        pygame.draw.circle(standing, COLORS['BLACK'], (13, 10), 1)
        pygame.draw.circle(standing, COLORS['BLACK'], (19, 10), 1)
        # Mustache
        pygame.draw.rect(standing, COLORS['BLACK'], (12, 14, 8, 2))
        
        self.sprites['standing'] = standing
        
        # Running sprites (simple variation)
        running1 = standing.copy()
        running2 = standing.copy()
        self.sprites['running'] = [running1, running2]
        
        # Jumping sprite
        jumping = standing.copy()
        self.sprites['jumping'] = jumping
    
    def update(self, platforms: List['Platform'], particle_system: ParticleSystem, sound_manager: SoundManager):
        # Handle invulnerability
        if self.invulnerable:
            self.invulnerable_timer -= 1
            if self.invulnerable_timer <= 0:
                self.invulnerable = False
        
        # Handle input
        keys = pygame.key.get_pressed()
        self.vel_x = 0
        
        if keys[pygame.K_LEFT] or keys[pygame.K_a]:
            self.vel_x = -self.config.player_speed
            self.facing_right = False
        if keys[pygame.K_RIGHT] or keys[pygame.K_d]:
            self.vel_x = self.config.player_speed
            self.facing_right = True
        if (keys[pygame.K_SPACE] or keys[pygame.K_UP] or keys[pygame.K_w]) and self.on_ground:
            self.vel_y = -self.config.jump_power
            self.on_ground = False
            sound_manager.play_sound('jump')
            particle_system.add_jump_effect(self.rect.centerx, self.rect.bottom)
            
        # Apply gravity
        self.vel_y += GRAVITY
        if self.vel_y > MAX_FALL_SPEED:
            self.vel_y = MAX_FALL_SPEED
            
        # Move horizontally
        self.rect.x += self.vel_x
        
        # Check horizontal collisions
        for platform in platforms:
            if self.rect.colliderect(platform.rect):
                if self.vel_x > 0:
                    self.rect.right = platform.rect.left
                elif self.vel_x < 0:
                    self.rect.left = platform.rect.right
                    
        # Move vertically
        self.rect.y += self.vel_y
        self.on_ground = False
        
        # Check vertical collisions
        for platform in platforms:
            if self.rect.colliderect(platform.rect):
                if self.vel_y > 0:
                    self.rect.bottom = platform.rect.top
                    self.vel_y = 0
                    self.on_ground = True
                elif self.vel_y < 0:
                    self.rect.top = platform.rect.bottom
                    self.vel_y = 0
                    
        # Keep player on screen horizontally
        if self.rect.left < 0:
            self.rect.left = 0
        if self.rect.right > SCREEN_WIDTH:
            self.rect.right = SCREEN_WIDTH
            
        # Update animation
        self.animation_timer += 1
        
        # Check if player fell off screen
        if self.rect.y > SCREEN_HEIGHT:
            self.take_damage()
            
    def take_damage(self):
        if not self.invulnerable:
            self.lives -= 1
            self.invulnerable = True
            self.invulnerable_timer = 120  # 2 seconds at 60 FPS
            self.respawn()
    
    def respawn(self):
        self.rect.x = self.spawn_x
        self.rect.y = self.spawn_y
        self.vel_x = 0
        self.vel_y = 0
        
    def draw(self, screen):
        # Get current sprite
        if not self.on_ground:
            sprite = self.sprites['jumping']
        elif abs(self.vel_x) > 0.1:
            frame = (self.animation_timer // 10) % 2
            sprite = self.sprites['running'][frame]
        else:
            sprite = self.sprites['standing']
        
        # Flip sprite if facing left
        if not self.facing_right:
            sprite = pygame.transform.flip(sprite, True, False)
        
        # Handle invulnerability flashing
        if self.invulnerable and (self.invulnerable_timer // 5) % 2:
            return  # Skip drawing for flashing effect
            
        screen.blit(sprite, self.rect)

class Platform:
    def __init__(self, x: int, y: int, width: int, height: int, platform_type: str = 'normal'):
        self.rect = pygame.Rect(x, y, width, height)
        self.type = platform_type
        self.create_sprite()
        
    def create_sprite(self):
        self.sprite = pygame.Surface((self.rect.width, self.rect.height))
        
        if self.type == 'normal':
            # Brick pattern
            self.sprite.fill(COLORS['BROWN'])
            # Add brick lines
            for y in range(0, self.rect.height, 20):
                pygame.draw.line(self.sprite, COLORS['DARK_BROWN'], (0, y), (self.rect.width, y), 2)
            for x in range(0, self.rect.width, 40):
                pygame.draw.line(self.sprite, COLORS['DARK_BROWN'], (x, 0), (x, self.rect.height), 2)
        elif self.type == 'metal':
            self.sprite.fill(COLORS['GRAY'])
            # Add metallic shine effect
            pygame.draw.rect(self.sprite, COLORS['SILVER'], (2, 2, self.rect.width-4, 4))
        
        # Add border
        pygame.draw.rect(self.sprite, COLORS['BLACK'], (0, 0, self.rect.width, self.rect.height), 2)
        
    def draw(self, screen):
        screen.blit(self.sprite, self.rect)

class Enemy:
    def __init__(self, x: int, y: int, enemy_type: str = 'goomba'):
        self.rect = pygame.Rect(x, y, 28, 28)
        self.vel_x = random.choice([-2, 2])
        self.start_x = x
        self.range = 120
        self.type = enemy_type
        self.animation_timer = 0
        self.create_sprites()
        
    def create_sprites(self):
        self.sprites = []
        
        if self.type == 'goomba':
            # Create goomba sprites
            for i in range(2):
                sprite = pygame.Surface((28, 28), pygame.SRCALPHA)
                # Body
                pygame.draw.ellipse(sprite, COLORS['DARK_BROWN'], (2, 8, 24, 18))
                # Eyes
                pygame.draw.circle(sprite, COLORS['WHITE'], (10, 12), 3)
                pygame.draw.circle(sprite, COLORS['WHITE'], (18, 12), 3)
                pygame.draw.circle(sprite, COLORS['BLACK'], (10, 12), 1)
                pygame.draw.circle(sprite, COLORS['BLACK'], (18, 12), 1)
                # Feet (animated)
                foot_offset = i * 2
                pygame.draw.ellipse(sprite, COLORS['BLACK'], (4 + foot_offset, 22, 6, 4))
                pygame.draw.ellipse(sprite, COLORS['BLACK'], (18 - foot_offset, 22, 6, 4))
                
                self.sprites.append(sprite)
        
    def update(self, platforms: List[Platform]):
        self.rect.x += self.vel_x
        
        # Reverse direction at range limits
        if abs(self.rect.x - self.start_x) > self.range:
            self.vel_x *= -1
            
        # Apply gravity and platform collision
        self.rect.y += 5
        for platform in platforms:
            if self.rect.colliderect(platform.rect):
                if self.rect.bottom > platform.rect.top:
                    self.rect.bottom = platform.rect.top
                    break
        
        # Update animation
        self.animation_timer += 1
                    
    def draw(self, screen):
        frame = (self.animation_timer // 15) % len(self.sprites)
        sprite = self.sprites[frame]
        
        # Flip sprite based on direction
        if self.vel_x < 0:
            sprite = pygame.transform.flip(sprite, True, False)
            
        screen.blit(sprite, self.rect)

class Coin:
    def __init__(self, x: int, y: int):
        self.rect = pygame.Rect(x, y, 24, 24)
        self.collected = False
        self.rotation = 0
        self.float_offset = 0
        self.original_y = y
        
    def update(self):
        if not self.collected:
            self.rotation += 8
            self.float_offset += 0.15
            self.rect.y = self.original_y + math.sin(self.float_offset) * 3
        
    def draw(self, screen):
        if not self.collected:
            # Create coin with rotation effect
            coin_surface = pygame.Surface((24, 24), pygame.SRCALPHA)
            
            # Calculate width based on rotation for 3D effect
            width = max(4, int(24 * abs(math.cos(math.radians(self.rotation)))))
            
            # Draw coin
            pygame.draw.ellipse(coin_surface, COLORS['GOLD'], 
                              (12 - width//2, 2, width, 20))
            pygame.draw.ellipse(coin_surface, COLORS['YELLOW'], 
                              (12 - width//2 + 2, 4, max(1, width-4), 16))
            
            # Add shine effect
            if width > 8:
                pygame.draw.ellipse(coin_surface, COLORS['WHITE'], 
                                  (12 - width//2 + 1, 6, max(1, width//3), 4))
            
            screen.blit(coin_surface, self.rect)

class PowerUp:
    def __init__(self, x: int, y: int, power_type: str):
        self.rect = pygame.Rect(x, y, 32, 32)
        self.type = power_type
        self.collected = False
        self.animation_timer = 0
        
    def update(self):
        self.animation_timer += 1
        
    def draw(self, screen):
        if not self.collected:
            if self.type == 'mushroom':
                # Red mushroom with white spots
                pygame.draw.ellipse(screen, COLORS['RED'], self.rect)
                pygame.draw.circle(screen, COLORS['WHITE'], 
                                 (self.rect.x + 8, self.rect.y + 8), 4)
                pygame.draw.circle(screen, COLORS['WHITE'], 
                                 (self.rect.x + 24, self.rect.y + 12), 3)
                pygame.draw.circle(screen, COLORS['WHITE'], 
                                 (self.rect.x + 16, self.rect.y + 20), 3)

class Goal:
    def __init__(self, x: int, y: int):
        self.rect = pygame.Rect(x, y, 48, 80)
        self.flag_animation = 0
        
    def update(self):
        self.flag_animation += 1
        
    def draw(self, screen):
        # Draw flag pole
        pygame.draw.rect(screen, COLORS['GRAY'], (self.rect.x + 22, self.rect.y, 4, 80))
        
        # Animated flag
        flag_wave = math.sin(self.flag_animation * 0.2) * 2
        flag_points = [
            (self.rect.x + 26, self.rect.y + 8),
            (self.rect.x + 46 + flag_wave, self.rect.y + 18),
            (self.rect.x + 26, self.rect.y + 28)
        ]
        pygame.draw.polygon(screen, COLORS['RED'], flag_points)
        
        # Flag pole base
        pygame.draw.rect(screen, COLORS['DARK_BROWN'], (self.rect.x + 18, self.rect.y + 76, 12, 8))

class UI:
    def __init__(self):
        self.font_small = pygame.font.Font(None, 24)
        self.font_medium = pygame.font.Font(None, 36)
        self.font_large = pygame.font.Font(None, 48)
        self.font_title = pygame.font.Font(None, 72)
        
    def draw_game_ui(self, screen, player: Player, level: int, coins_collected: int, total_coins: int):
        # Create UI background
        ui_surface = pygame.Surface((SCREEN_WIDTH, 60), pygame.SRCALPHA)
        ui_surface.fill((*COLORS['BLACK'], 128))
        screen.blit(ui_surface, (0, 0))
        
        # Score
        score_text = self.font_medium.render(f"Score: {player.score:06d}", True, COLORS['WHITE'])
        screen.blit(score_text, (20, 15))
        
        # Lives with heart icons
        lives_text = self.font_medium.render("Lives:", True, COLORS['WHITE'])
        screen.blit(lives_text, (220, 15))
        for i in range(player.lives):
            heart_x = 290 + i * 25
            pygame.draw.circle(screen, COLORS['RED'], (heart_x, 30), 8)
            pygame.draw.circle(screen, COLORS['RED'], (heart_x + 10, 30), 8)
            pygame.draw.polygon(screen, COLORS['RED'], [
                (heart_x - 6, 35), (heart_x + 5, 45), (heart_x + 16, 35)
            ])
        
        # Level
        level_text = self.font_medium.render(f"Level: {level}", True, COLORS['WHITE'])
        screen.blit(level_text, (450, 15))
        
        # Coins with icon
        coin_surface = pygame.Surface((20, 20), pygame.SRCALPHA)
        pygame.draw.circle(coin_surface, COLORS['GOLD'], (10, 10), 8)
        pygame.draw.circle(coin_surface, COLORS['YELLOW'], (10, 10), 6)
        screen.blit(coin_surface, (SCREEN_WIDTH - 150, 20))
        
        coins_text = self.font_medium.render(f"{coins_collected}/{total_coins}", True, COLORS['WHITE'])
        screen.blit(coins_text, (SCREEN_WIDTH - 120, 15))
    
    def draw_menu(self, screen):
        # Background gradient
        for y in range(SCREEN_HEIGHT):
            color_ratio = y / SCREEN_HEIGHT
            color = (
                int(135 * (1 - color_ratio) + 206 * color_ratio),
                int(206 * (1 - color_ratio) + 235 * color_ratio),
                int(235 * (1 - color_ratio) + 255 * color_ratio)
            )
            pygame.draw.line(screen, color, (0, y), (SCREEN_WIDTH, y))
        
        # Title
        title_text = self.font_title.render("SUPER MARIO", True, COLORS['RED'])
        title_shadow = self.font_title.render("SUPER MARIO", True, COLORS['BLACK'])
        
        title_rect = title_text.get_rect(center=(SCREEN_WIDTH//2, 200))
        shadow_rect = title_rect.copy()
        shadow_rect.x += 3
        shadow_rect.y += 3
        
        screen.blit(title_shadow, shadow_rect)
        screen.blit(title_text, title_rect)
        
        # Subtitle
        subtitle_text = self.font_large.render("Python Edition", True, COLORS['BLUE'])
        subtitle_rect = subtitle_text.get_rect(center=(SCREEN_WIDTH//2, 280))
        screen.blit(subtitle_text, subtitle_rect)
        
        # Instructions
        instructions = [
            "Press SPACE or ENTER to Start",
            "Arrow Keys or WASD to Move",
            "ESC to Pause",
            "Collect coins and reach the flag!"
        ]
        
        for i, instruction in enumerate(instructions):
            text = self.font_medium.render(instruction, True, COLORS['BLACK'])
            text_rect = text.get_rect(center=(SCREEN_WIDTH//2, 400 + i * 40))
            screen.blit(text, text_rect)

class Game:
    def __init__(self):
        self.screen = pygame.display.set_mode((SCREEN_WIDTH, SCREEN_HEIGHT))
        pygame.display.set_caption("Super Mario Python - Professional Edition")
        self.clock = pygame.time.Clock()
        
        # Game state
        self.state = GameState.MENU
        self.current_level = 1
        self.max_levels = 4
        
        # Game systems
        self.config = GameConfig()
        self.sound_manager = SoundManager()
        self.particle_system = ParticleSystem()
        self.ui = UI()
        
        # Game objects
        self.player = None
        self.platforms = []
        self.enemies = []
        self.coins = []
        self.power_ups = []
        self.goal = None
        
        # Performance tracking
        self.frame_count = 0
        self.fps_display = True
        
    def setup_level(self):
        """Initialize the current level with all game objects"""
        spawn_x, spawn_y = 50, 400
        self.player = Player(spawn_x, spawn_y, self.config)
        self.platforms = []
        self.enemies = []
        self.coins = []
        self.power_ups = []
        self.goal = None
        
        if self.current_level == 1:
            self._setup_level_1()
        elif self.current_level == 2:
            self._setup_level_2()
        elif self.current_level == 3:
            self._setup_level_3()
        elif self.current_level == 4:
            self._setup_level_4()
    
    def _setup_level_1(self):
        """Tutorial level - Easy introduction"""
        # Platforms
        self.platforms = [
            Platform(0, 500, 300, 20),
            Platform(400, 450, 150, 20),
            Platform(650, 400, 100, 20),
            Platform(800, 350, 200, 20),
            Platform(300, 350, 80, 20),
            Platform(200, 280, 120, 20),
            Platform(500, 300, 100, 20),
            Platform(0, 650, SCREEN_WIDTH, 50)  # Ground
        ]
        
        # Enemies
        self.enemies = [
            Enemy(420, 422, 'goomba'),
            Enemy(220, 252, 'goomba')
        ]
        
        # Coins
        coin_positions = [
            (450, 420), (480, 420), (510, 420),
            (680, 370), (230, 250), (330, 320),
            (520, 270), (850, 320)
        ]
        self.coins = [Coin(x, y) for x, y in coin_positions]
        
        # Power-ups
        self.power_ups = [PowerUp(540, 270, 'mushroom')]
        
        # Goal
        self.goal = Goal(1120, 270)
    
    def _setup_level_2(self):
        """Moderate difficulty with more platforming"""
        self.platforms = [
            Platform(0, 500, 200, 20),
            Platform(300, 480, 100, 20),
            Platform(500, 420, 80, 20),
            Platform(650, 380, 120, 20),
            Platform(150, 350, 100, 20),
            Platform(400, 300, 80, 20),
            Platform(600, 250, 100, 20),
            Platform(800, 200, 200, 20),
            Platform(200, 200, 80, 20),
            Platform(0, 650, SCREEN_WIDTH, 50)
        ]
        
        self.enemies = [
            Enemy(320, 452, 'goomba'),
            Enemy(670, 352, 'goomba'),
            Enemy(820, 172, 'goomba'),
            Enemy(220, 172, 'goomba')
        ]
        
        coin_positions = [
            (330, 450), (360, 450), (520, 390),
            (680, 350), (170, 320), (420, 270),
            (620, 220), (850, 170), (230, 170),
            (920, 170)
        ]
        self.coins = [Coin(x, y) for x, y in coin_positions]
        
        self.power_ups = [
            PowerUp(170, 320, 'mushroom'),
            PowerUp(640, 220, 'mushroom')
        ]
        
        self.goal = Goal(1120, 120)
    
    def _setup_level_3(self):
        """Vertical challenge with precise jumping"""
        self.platforms = [
            Platform(0, 500, 150, 20),
            Platform(200, 450, 80, 20),
            Platform(350, 400, 80, 20),
            Platform(500, 350, 80, 20),
            Platform(650, 300, 80, 20),
            Platform(150, 300, 80, 20),
            Platform(300, 250, 80, 20),
            Platform(450, 200, 80, 20),
            Platform(600, 150, 80, 20),
            Platform(750, 100, 80, 20),
            Platform(900, 50, 300, 20),
            Platform(0, 650, SCREEN_WIDTH, 50)
        ]
        
        self.enemies = [
            Enemy(220, 422, 'goomba'),
            Enemy(520, 322, 'goomba'),
            Enemy(670, 272, 'goomba'),
            Enemy(170, 272, 'goomba'),
            Enemy(470, 172, 'goomba'),
            Enemy(770, 72, 'goomba'),
            Enemy(950, 22, 'goomba')
        ]
        
        coin_positions = [
            (230, 420), (380, 370), (530, 320),
            (680, 270), (180, 270), (330, 220),
            (480, 170), (630, 120), (780, 70),
            (950, 20), (1050, 20), (1150, 20)
        ]
        self.coins = [Coin(x, y) for x, y in coin_positions]
        
        self.power_ups = [
            PowerUp(330, 220, 'mushroom'),
            PowerUp(780, 70, 'mushroom')
        ]
        
        self.goal = Goal(1150, -10)
    
    def _setup_level_4(self):
        """Final challenge - Expert level"""
        self.platforms = [
            Platform(0, 480, 100, 20),
            Platform(150, 420, 60, 20),
            Platform(250, 380, 60, 20),
            Platform(350, 340, 60, 20),
            Platform(450, 300, 60, 20),
            Platform(100, 320, 80, 20),
            Platform(230, 280, 80, 20),
            Platform(360, 240, 80, 20),
            Platform(490, 200, 80, 20),
            Platform(620, 160, 80, 20),
            Platform(750, 120, 60, 20),
            Platform(550, 120, 60, 20),
            Platform(400, 80, 60, 20),
            Platform(250, 80, 60, 20),
            Platform(850, 80, 150, 20),
            Platform(1000, 40, 200, 20),
            Platform(0, 650, SCREEN_WIDTH, 50)
        ]
        
        self.enemies = [
            Enemy(170, 392, 'goomba'),
            Enemy(270, 352, 'goomba'),
            Enemy(370, 312, 'goomba'),
            Enemy(470, 272, 'goomba'),
            Enemy(120, 292, 'goomba'),
            Enemy(250, 252, 'goomba'),
            Enemy(380, 212, 'goomba'),
            Enemy(510, 172, 'goomba'),
            Enemy(640, 132, 'goomba'),
            Enemy(570, 92, 'goomba'),
            Enemy(870, 52, 'goomba'),
            Enemy(1050, 12, 'goomba')
        ]
        
        coin_positions = [
            (170, 390), (270, 350), (370, 310),
            (470, 270), (130, 290), (260, 250),
            (390, 210), (520, 170), (650, 130),
            (770, 90), (570, 90), (420, 50),
            (270, 50), (880, 50), (1080, 10),
            (1150, 10)
        ]
        self.coins = [Coin(x, y) for x, y in coin_positions]
        
        self.power_ups = [
            PowerUp(260, 250, 'mushroom'),
            PowerUp(570, 90, 'mushroom'),
            PowerUp(880, 50, 'mushroom')
        ]
        
        self.goal = Goal(1150, -20)
    
    def handle_collisions(self):
        """Handle all collision detection"""
        # Enemy collisions
        for enemy in self.enemies[:]:
            if self.player.rect.colliderect(enemy.rect) and not self.player.invulnerable:
                self.player.take_damage()
                self.sound_manager.play_sound('game_over')
                if self.player.lives <= 0:
                    self.state = GameState.GAME_OVER
                    
        # Coin collisions
        for coin in self.coins:
            if not coin.collected and self.player.rect.colliderect(coin.rect):
                coin.collected = True
                self.player.score += self.config.coin_value
                self.sound_manager.play_sound('coin')
                self.particle_system.add_coin_effect(coin.rect.centerx, coin.rect.centery)
                
        # Power-up collisions
        for power_up in self.power_ups:
            if not power_up.collected and self.player.rect.colliderect(power_up.rect):
                power_up.collected = True
                self.player.score += 200
                # Add power-up effect (could extend player abilities)
                
        # Goal collision
        if self.goal and self.player.rect.colliderect(self.goal.rect):
            if self.current_level < self.max_levels:
                self.current_level += 1
                self.player.score += self.config.level_bonus
                self.setup_level()
            else:
                self.state = GameState.VICTORY
    
    def handle_events(self):
        """Handle all pygame events"""
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                return False
                
            elif event.type == pygame.KEYDOWN:
                if self.state == GameState.MENU:
                    if event.key in [pygame.K_SPACE, pygame.K_RETURN]:
                        self.state = GameState.PLAYING
                        self.setup_level()
                        
                elif self.state == GameState.PLAYING:
                    if event.key == pygame.K_ESCAPE:
                        self.state = GameState.PAUSED
                    elif event.key == pygame.K_F1:
                        self.fps_display = not self.fps_display
                        
                elif self.state == GameState.PAUSED:
                    if event.key == pygame.K_ESCAPE:
                        self.state = GameState.PLAYING
                    elif event.key == pygame.K_r:
                        self.restart_game()
                        
                elif self.state in [GameState.GAME_OVER, GameState.VICTORY]:
                    if event.key == pygame.K_r:
                        self.restart_game()
                    elif event.key == pygame.K_ESCAPE:
                        self.state = GameState.MENU
                        
        return True
    
    def restart_game(self):
        """Restart the game from level 1"""
        self.current_level = 1
        self.state = GameState.PLAYING
        self.setup_level()
        self.particle_system.particles.clear()
    
    def update_game(self):
        """Update all game objects"""
        if self.state == GameState.PLAYING:
            # Update player
            self.player.update(self.platforms, self.particle_system, self.sound_manager)
            
            # Update enemies
            for enemy in self.enemies:
                enemy.update(self.platforms)
                
            # Update coins
            for coin in self.coins:
                coin.update()
                
            # Update power-ups
            for power_up in self.power_ups:
                power_up.update()
                
            # Update goal
            if self.goal:
                self.goal.update()
                
            # Update particle system
            self.particle_system.update()
            
            # Handle collisions
            self.handle_collisions()
    
    def draw_game(self):
        """Draw all game objects"""
        # Background gradient
        for y in range(SCREEN_HEIGHT):
            color_ratio = y / SCREEN_HEIGHT
            color = (
                int(135 * (1 - color_ratio) + 200 * color_ratio),
                int(206 * (1 - color_ratio) + 230 * color_ratio),
                int(235 * (1 - color_ratio) + 255 * color_ratio)
            )
            pygame.draw.line(self.screen, color, (0, y), (SCREEN_WIDTH, y))
        
        # Draw platforms
        for platform in self.platforms:
            platform.draw(self.screen)
            
        # Draw enemies
        for enemy in self.enemies:
            enemy.draw(self.screen)
            
        # Draw coins
        for coin in self.coins:
            coin.draw(self.screen)
            
        # Draw power-ups
        for power_up in self.power_ups:
            power_up.draw(self.screen)
            
        # Draw goal
        if self.goal:
            self.goal.draw(self.screen)
            
        # Draw particles
        self.particle_system.draw(self.screen)
        
        # Draw player
        if self.player:
            self.player.draw(self.screen)
            
        # Draw UI
        if self.player:
            coins_collected = sum(1 for coin in self.coins if coin.collected)
            total_coins = len(self.coins)
            self.ui.draw_game_ui(self.screen, self.player, self.current_level, 
                               coins_collected, total_coins)
    
    def draw_pause_screen(self):
        """Draw pause overlay"""
        # Semi-transparent overlay
        overlay = pygame.Surface((SCREEN_WIDTH, SCREEN_HEIGHT), pygame.SRCALPHA)
        overlay.fill((*COLORS['BLACK'], 128))
        self.screen.blit(overlay, (0, 0))
        
        # Pause text
        pause_text = self.ui.font_title.render("PAUSED", True, COLORS['WHITE'])
        pause_rect = pause_text.get_rect(center=(SCREEN_WIDTH//2, SCREEN_HEIGHT//2 - 50))
        self.screen.blit(pause_text, pause_rect)
        
        # Instructions
        instructions = [
            "ESC - Resume Game",
            "R - Restart Level",
            "F1 - Toggle FPS Display"
        ]
        
        for i, instruction in enumerate(instructions):
            text = self.ui.font_medium.render(instruction, True, COLORS['WHITE'])
            text_rect = text.get_rect(center=(SCREEN_WIDTH//2, SCREEN_HEIGHT//2 + 20 + i * 40))
            self.screen.blit(text, text_rect)
    
    def draw_game_over_screen(self):
        """Draw game over screen"""
        # Background
        overlay = pygame.Surface((SCREEN_WIDTH, SCREEN_HEIGHT), pygame.SRCALPHA)
        overlay.fill((*COLORS['RED'], 100))
        self.screen.blit(overlay, (0, 0))
        
        # Game over text
        game_over_text = self.ui.font_title.render("GAME OVER", True, COLORS['RED'])
        game_over_rect = game_over_text.get_rect(center=(SCREEN_WIDTH//2, SCREEN_HEIGHT//2 - 100))
        self.screen.blit(game_over_text, game_over_rect)
        
        # Score
        if self.player:
            score_text = self.ui.font_large.render(f"Final Score: {self.player.score:06d}", True, COLORS['WHITE'])
            score_rect = score_text.get_rect(center=(SCREEN_WIDTH//2, SCREEN_HEIGHT//2 - 20))
            self.screen.blit(score_text, score_rect)
        
        # Instructions
        restart_text = self.ui.font_medium.render("R - Restart Game", True, COLORS['WHITE'])
        restart_rect = restart_text.get_rect(center=(SCREEN_WIDTH//2, SCREEN_HEIGHT//2 + 40))
        self.screen.blit(restart_text, restart_rect)
        
        menu_text = self.ui.font_medium.render("ESC - Main Menu", True, COLORS['WHITE'])
        menu_rect = menu_text.get_rect(center=(SCREEN_WIDTH//2, SCREEN_HEIGHT//2 + 80))
        self.screen.blit(menu_text, menu_rect)
    
    def draw_victory_screen(self):
        """Draw victory screen"""
        # Background
        overlay = pygame.Surface((SCREEN_WIDTH, SCREEN_HEIGHT), pygame.SRCALPHA)
        overlay.fill((*COLORS['GOLD'], 100))
        self.screen.blit(overlay, (0, 0))
        
        # Victory text with animation
        scale = 1.0 + 0.1 * math.sin(self.frame_count * 0.1)
        victory_font = pygame.font.Font(None, int(72 * scale))
        victory_text = victory_font.render("VICTORY!", True, COLORS['GOLD'])
        victory_rect = victory_text.get_rect(center=(SCREEN_WIDTH//2, SCREEN_HEIGHT//2 - 100))
        self.screen.blit(victory_text, victory_rect)
        
        # Congratulations
        congrats_text = self.ui.font_large.render("Congratulations! You completed all levels!", True, COLORS['WHITE'])
        congrats_rect = congrats_text.get_rect(center=(SCREEN_WIDTH//2, SCREEN_HEIGHT//2 - 40))
        self.screen.blit(congrats_text, congrats_rect)
        
        # Final score
        if self.player:
            score_text = self.ui.font_large.render(f"Final Score: {self.player.score:06d}", True, COLORS['WHITE'])
            score_rect = score_text.get_rect(center=(SCREEN_WIDTH//2, SCREEN_HEIGHT//2 + 20))
            self.screen.blit(score_text, score_rect)
        
        # Instructions
        restart_text = self.ui.font_medium.render("R - Play Again", True, COLORS['WHITE'])
        restart_rect = restart_text.get_rect(center=(SCREEN_WIDTH//2, SCREEN_HEIGHT//2 + 80))
        self.screen.blit(restart_text, restart_rect)
        
        menu_text = self.ui.font_medium.render("ESC - Main Menu", True, COLORS['WHITE'])
        menu_rect = menu_text.get_rect(center=(SCREEN_WIDTH//2, SCREEN_HEIGHT//2 + 120))
        self.screen.blit(menu_text, menu_rect)
    
    def draw_fps(self):
        """Draw FPS counter"""
        if self.fps_display:
            fps = self.clock.get_fps()
            fps_text = self.ui.font_small.render(f"FPS: {fps:.1f}", True, COLORS['WHITE'])
            fps_bg = pygame.Surface((80, 25), pygame.SRCALPHA)
            fps_bg.fill((*COLORS['BLACK'], 128))
            self.screen.blit(fps_bg, (SCREEN_WIDTH - 85, SCREEN_HEIGHT - 30))
            self.screen.blit(fps_text, (SCREEN_WIDTH - 80, SCREEN_HEIGHT - 25))
    
    def run(self):
        """Main game loop"""
        running = True
        
        while running:
            self.frame_count += 1
            
            # Handle events
            running = self.handle_events()
            
            # Update game
            self.update_game()
            
            # Draw everything
            if self.state == GameState.MENU:
                self.ui.draw_menu(self.screen)
            elif self.state == GameState.PLAYING:
                self.draw_game()
            elif self.state == GameState.PAUSED:
                self.draw_game()
                self.draw_pause_screen()
            elif self.state == GameState.GAME_OVER:
                self.draw_game_over_screen()
            elif self.state == GameState.VICTORY:
                self.draw_victory_screen()
            
            # Draw FPS counter
            self.draw_fps()
            
            # Update display
            pygame.display.flip()
            self.clock.tick(FPS)
        
        pygame.quit()
        sys.exit()

def main():
    """Entry point of the game"""
    try:
        game = Game()
        game.run()
    except Exception as e:
        print(f"Error running game: {e}")
        pygame.quit()
        sys.exit(1)

if __name__ == "__main__":
    main()# mario
