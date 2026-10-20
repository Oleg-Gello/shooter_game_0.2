# shooter_game_0.2
игра шутер
from pygame import *
import random

init()
mixer.init()

try:
    mixer.music.load("space.ogg")
    mixer.music.play(-1)
    fire = mixer.Sound("fire.ogg")
    collect = mixer.Sound("lound_collect.mp3")
    reload_sound = mixer.Sound("reload.ogg")
except:
    fire = None
    collect = None
    reload_sound = None

font.init()
font1 = font.SysFont("Arial", 36)
font2 = font.SysFont("Arial", 72)
font3 = font.SysFont("Arial", 48)
font_menu_title = font.SysFont("Arial", 64)
font_button = font.SysFont("Arial", 40)

win_text = font2.render('YOU WIN!', True, (0, 255, 0))
lose_text = font2.render('YOU LOSE!', True, (255, 0, 0))
next_level_text = font3.render('NEXT LEVEL', True, (255, 255, 0))
restart_text = font3.render('RESTART', True, (255, 255, 0))

window = display.set_mode((700, 500))
display.set_caption('Shooter_Space')

try:
    background = transform.scale(image.load('galaxy.jpg'), (700, 500))
    rocket_img = 'rocket.png'
    enemy_img = 'ufo.png'
    bullet_img = 'bullet.png'
    asteroid_img = 'asteroid.png'
    bonus_fast_bullet = 'bonus_fast_bullet.png'
    bonus_freeze = 'bonus_freeze.png'
    bonus_rapid_fire = 'bonus_rapid_fire.png'
    bonus_extra_life = 'bonus_extra_life.png'
    bonus_double_shot = 'bonus_double_shot.png'
except:
    background = Surface((700, 500)); background.fill((0, 0, 20))
    rocket_img = None
    enemy_img = None
    bullet_img = None
    asteroid_img = None
    bonus_fast_bullet = None
    bonus_freeze = None
    bonus_rapid_fire = None
    bonus_extra_life = None
    bonus_double_shot = None

clock = time.Clock()
FPS = 60

levels = [
    {'enemies_count': 5,  'speed_min': 1, 'speed_max': 2, 'score_goal': 8,  'lost_limit': 3, 'max_ammo': 10, 'asteroids_count': 1},
    {'enemies_count': 8,  'speed_min': 2, 'speed_max': 3, 'score_goal': 12, 'lost_limit': 3, 'max_ammo': 8,  'asteroids_count': 2},
    {'enemies_count': 10, 'speed_min': 2, 'speed_max': 4, 'score_goal': 15, 'lost_limit': 3, 'max_ammo': 7,  'asteroids_count': 2},
    {'enemies_count': 15, 'speed_min': 3, 'speed_max': 5, 'score_goal': 20, 'lost_limit': 2, 'max_ammo': 6,  'asteroids_count': 3},
    {'enemies_count': 20, 'speed_min': 4, 'speed_max': 6, 'score_goal': 25, 'lost_limit': 2, 'max_ammo': 5,  'asteroids_count': 4},
]

score = 0
lost = 0
player_lives = 3
game = True
finish = False
current_level = 0
RELOAD_TIME = 3000

# ----- Громкость и меню -----
volume = 0.5  # от 0 до 1
dragging_slider = False

def apply_volume():
    mixer.music.set_volume(volume)
    for snd in (fire, collect, reload_sound):
        if snd:
            snd.set_volume(volume)

apply_volume()


class Button:
    def __init__(self, x, y, w, h, text):
        self.rect = Rect(x, y, w, h)
        self.text = font_button.render(text, True, (255, 255, 255))

    def draw(self, surface):
        mouse_pos = mouse.get_pos()
        hovered = self.rect.collidepoint(mouse_pos)
        color = (60, 60, 120) if hovered else (30, 30, 70)
        draw.rect(surface, color, self.rect, border_radius=12)
        draw.rect(surface, (100, 100, 200), self.rect, 3, border_radius=12)
        surface.blit(self.text, (self.rect.centerx - self.text.get_width() // 2,
                                 self.rect.centery - self.text.get_height() // 2))

    def is_clicked(self, pos):
        return self.rect.collidepoint(pos)


class GameSprite(sprite.Sprite):
    def __init__(self, player_image, player_x, player_y, player_speed, w=76, h=76):
        super().__init__()
        if player_image:
            self.image = transform.scale(image.load(player_image), (w, h))
        else:
            self.image = Surface((w, h)); self.image.fill((255, 255, 255))
        self.speed = player_speed
        self.rect = self.image.get_rect()
        self.rect.x = player_x
        self.rect.y = player_y

    def reset(self):
        window.blit(self.image, (self.rect.x, self.rect.y))


class Player(GameSprite):
    def __init__(self, player_image, player_x, player_y, player_speed, w=90, h=90):
        super().__init__(player_image, player_x, player_y, player_speed, w, h)
        self.bullet_speed_multiplier = 1.0
        self.fire_rate_multiplier = 1.0
        self.last_shot_time = 0
        self.fast_bullet_end_time = 0
        self.rapid_fire_end_time = 0
        self.double_shot_active = False
        self.double_shot_end_time = 0
        self.ammo = 10
        self.max_ammo = 10
        self.reloading = False
        self.reload_end_time = 0

    def update(self):
        keys_pressed = key.get_pressed()
        if keys_pressed[K_LEFT] and self.rect.x > 0:
            self.rect.x -= self.speed
        if keys_pressed[K_RIGHT] and self.rect.x < 700 - self.rect.width:
            self.rect.x += self.speed
        now = time.get_ticks()
        if self.reloading and now >= self.reload_end_time:
            self.reloading = False
            self.ammo = self.max_ammo

    def start_reload(self):
        self.reloading = True
        self.reload_end_time = time.get_ticks() + RELOAD_TIME
        if reload_sound:
            reload_sound.play()

    def fire(self):
        if self.reloading:
            return
        if self.ammo <= 0:
            self.start_reload()
            return
        current_time = time.get_ticks()
        cooldown = 300 / self.fire_rate_multiplier
        if current_time - self.last_shot_time >= cooldown:
            bullet_speed = int(15 * self.bullet_speed_multiplier)
            bullet = Bullet(bullet_img, self.rect.centerx - 8, self.rect.top, bullet_speed, w=16, h=16)
            bullets.add(bullet)
            if self.double_shot_active:
                bullet2 = Bullet(bullet_img, self.rect.centerx + 8, self.rect.top, bullet_speed, w=16, h=16)
                bullets.add(bullet2)
            self.ammo -= 1
            self.last_shot_time = current_time
            if fire:
                fire.play()
            if self.ammo <= 0:
                self.start_reload()


class Enemy(GameSprite):
    def __init__(self, player_image, player_x, player_y, player_speed, w=90, h=90):
        super().__init__(player_image, player_x, player_y, player_speed, w, h)
        self.frozen = False
        self.freeze_end_time = 0

    def update(self):
        global lost
        now = time.get_ticks()
        if self.frozen and now < self.freeze_end_time:
            pass
        else:
            self.frozen = False
            self.rect.y += self.speed
            if self.rect.y > 500:
                lost += 1
                self.respawn()

    def respawn(self):
        self.rect.x = random.randint(0, 700 - self.rect.width)
        self.rect.y = -self.rect.height


class Bullet(GameSprite):
    def __init__(self, player_image, player_x, player_y, player_speed, w=18, h=18):
        super().__init__(player_image, player_x, player_y, player_speed, w, h)

    def update(self):
        self.rect.y -= self.speed
        if self.rect.y < 0:
            self.kill()


class Asteroid(GameSprite):
    def __init__(self, player_image, player_x, player_y, player_speed, w=50, h=50):
        super().__init__(player_image, player_x, player_y, player_speed, w, h)

    def update(self):
        self.rect.y += self.speed
        if self.rect.y > 500:
            self.rect.x = random.randint(0, 700 - self.rect.width)
            self.rect.y = -self.rect.height


class Bonus(GameSprite):
    def __init__(self, bonus_type, player_image, player_x, player_y, player_speed=3, w=40, h=40):
        super().__init__(player_image, player_x, player_y, player_speed, w, h)
        self.bonus_type = bonus_type

    def update(self):
        self.rect.y += self.speed
        if self.rect.y > 500:
            self.kill()


player = Player(rocket_img, 300, 400, 5)
enemies = sprite.Group()
bullets = sprite.Group()
bonuses = sprite.Group()
asteroids = sprite.Group()
spawn_timer = 0
spawn_interval = 3000


def create_enemies(level_data):
    enemies.empty()
    for _ in range(level_data['enemies_count']):
        speed = random.randint(level_data['speed_min'], level_data['speed_max'])
        enemy = Enemy(enemy_img, random.randint(0, 700 - 90), -random.randint(80, 500), speed)
        enemies.add(enemy)


def create_asteroids(level_data):
    asteroids.empty()
    for _ in range(level_data.get('asteroids_count', 0)):
        speed = random.randint(1, 3)
        ast = Asteroid(asteroid_img, random.randint(0, 700 - 50), -random.randint(0, 400), speed, w=50, h=50)
        asteroids.add(ast)


def reset_game():
    global score, lost, current_level, finish, spawn_timer, spawn_interval, player_lives
    score = 0
    lost = 0
    player_lives = 3
    current_level = 0
    finish = False
    spawn_timer = 0
    spawn_interval = 3000
    bonuses.empty()
    create_enemies(levels[current_level])
    create_asteroids(levels[current_level])
    player.max_ammo = levels[current_level]['max_ammo']
    player.ammo = player.max_ammo
    player.reloading = False
    player.rect.x = 300
    player.rect.y = 400


def spawn_random_bonus():
    types = [
        ('fast_bullet', bonus_fast_bullet),
        ('freeze', bonus_freeze),
        ('rapid_fire', bonus_rapid_fire),
        ('extra_life', bonus_extra_life),
        ('double_shot', bonus_double_shot),
    ]
    bonus_type, img = random.choice(types)
    start_x = random.randint(0, 700 - 40)
    bonus = Bonus(bonus_type, img, start_x, -40)
    bonuses.add(bonus)


def load_level(idx):
    global score, lost, finish, spawn_timer, spawn_interval, player_lives
    lvl = levels[idx]
    score = 0
    lost = 0
    finish = False
    spawn_timer = 0
    spawn_interval = 3000
    bonuses.empty()
    bullets.empty()
    create_enemies(lvl)
    create_asteroids(lvl)
    player.max_ammo = lvl['max_ammo']
    player.ammo = player.max_ammo
    player.reloading = False
    player.rect.x = 300
    player.rect.y = 400
    player_lives = 3


# ----- Кнопки меню и настройки -----
play_button = Button(250, 250, 200, 70, 'Играть')
settings_button = Button(250, 350, 200, 70, 'Настройки')
back_button = Button(250, 400, 200, 70, 'Назад')

# Ползунок громкости
slider_x, slider_y, slider_w, slider_h = 200, 280, 300, 10
slider_knob = Rect(0, slider_y - 10, 20, 30)


def draw_slider():
    global volume
    # Дорожка
    draw.rect(window, (100, 100, 100), (slider_x, slider_y, slider_w, slider_h), border_radius=5)
    # Заполненная часть
    fill_w = int(slider_w * volume)
    draw.rect(window, (0, 200, 100), (slider_x, slider_y, fill_w, slider_h), border_radius=5)
    # Кружок
    slider_knob.centerx = slider_x + fill_w
    slider_knob.centery = slider_y + slider_h // 2
    draw.circle(window, (255, 255, 255), slider_knob.center, 12)
    draw.circle(window, (0, 150, 255), slider_knob.center, 12, 3)
    # Подпись
    vol_text = font1.render(f'Громкость: {int(volume * 100)}%', True, (255, 255, 255))
    window.blit(vol_text, (slider_x, slider_y - 70))


def update_slider(pos_x):
    global volume
    volume = (pos_x - slider_x) / slider_w
    volume = max(0, min(1, volume))
    apply_volume()


create_enemies(levels[current_level])
create_asteroids(levels[current_level])
player.max_ammo = levels[current_level]['max_ammo']
player.ammo = player.max_ammo

# Состояния: 'menu' — главное меню, 'settings' — настройки, 'game' — игра
state = 'menu'

while game:
    now = time.get_ticks()

    for e in event.get():
        if e.type == QUIT:
            game = False

        elif state == 'menu':
            if e.type == MOUSEBUTTONDOWN and e.button == 1:
                if play_button.is_clicked(e.pos):
                    reset_game()
                    state = 'game'
                elif settings_button.is_clicked(e.pos):
                    state = 'settings'
            if e.type == KEYDOWN and e.key == K_ESCAPE:
                game = False

        elif state == 'settings':
            if e.type == MOUSEBUTTONDOWN and e.button == 1:
                if back_button.is_clicked(e.pos):
                    state = 'menu'
                # Клик по дорожке или кружку — двигаем ползунок
                elif slider_y - 15 <= e.pos[1] <= slider_y + slider_h + 15 and slider_x - 10 <= e.pos[0] <= slider_x + slider_w + 10:
                    dragging_slider = True
                    update_slider(e.pos[0])
            elif e.type == MOUSEMOTION and dragging_slider:
                update_slider(e.pos[0])
            elif e.type == MOUSEBUTTONUP:
                dragging_slider = False
            if e.type == KEYDOWN and e.key == K_ESCAPE:
                state = 'menu'

        elif state == 'game':
            if e.type == KEYDOWN:
                if e.key == K_SPACE and not finish:
                    player.fire()
                elif e.key == K_RETURN and finish and current_level < len(levels) - 1:
                    current_level += 1
                    load_level(current_level)
                elif e.key == K_ESCAPE and finish:
                    state = 'menu'   # выход в меню, а не закрытие игры
                elif e.key == K_r and finish:
                    reset_game()

    window.blit(background, (0, 0))

    if state == 'menu':
        title = font_menu_title.render('SHOOTER SPACE', True, (255, 255, 255))
        window.blit(title, (350 - title.get_width() // 2, 100))
        play_button.draw(window)
        settings_button.draw(window)

    elif state == 'settings':
        set_title = font_menu_title.render('Настройки', True, (255, 255, 255))
        window.blit(set_title, (350 - set_title.get_width() // 2, 120))
        draw_slider()
        back_button.draw(window)

    elif state == 'game':
        if not finish:
            if now - spawn_timer >= spawn_interval:
                spawn_random_bonus()
                spawn_timer = now
                spawn_interval = random.randint(2000, 5000)
            if player.reloading:
                ammo_display = 'ПЕРЕЗАРЯДКА...'
            else:
                ammo_display = f'Патроны: {player.ammo}/{player.max_ammo}'
            hud_text = font1.render(
                f'Lvl {current_level + 1} | Сбито: {score} | Пропущ: {lost} | Жизни: {player_lives} | {ammo_display}',
                True, (255, 255, 255))
            window.blit(hud_text, (10, 10))
            player.update()
            player.reset()
            enemies.update()
            enemies.draw(window)
            asteroids.update()
            asteroids.draw(window)
            bullets.update()
            bullets.draw(window)
            bonuses.update()
            bonuses.draw(window)

            hits_bonus = sprite.spritecollide(player, bonuses, True)
            for b in hits_bonus:
                if collect:
                    collect.play()
                if b.bonus_type == 'fast_bullet':
                    player.bullet_speed_multiplier = 2.0
                    player.fast_bullet_end_time = now + 10000
                elif b.bonus_type == 'freeze':
                    for enemy in enemies:
                        enemy.frozen = True
                        enemy.freeze_end_time = now + 5000
                elif b.bonus_type == 'rapid_fire':
                    player.fire_rate_multiplier = 2.0
                    player.rapid_fire_end_time = now + 10000
                elif b.bonus_type == 'extra_life':
                    player_lives += 1
                elif b.bonus_type == 'double_shot':
                    player.double_shot_active = True
                    player.double_shot_end_time = now + 10000

            if now >= player.fast_bullet_end_time and player.bullet_speed_multiplier > 1.0:
                player.bullet_speed_multiplier = 1.0
            if now >= player.rapid_fire_end_time and player.fire_rate_multiplier > 1.0:
                player.fire_rate_multiplier = 1.0
            if now >= player.double_shot_end_time and player.double_shot_active:
                player.double_shot_active = False

            hits = sprite.groupcollide(bullets, enemies, True, True)
            for _ in hits:
                score += 1
                lvl = levels[current_level]
                speed = random.randint(lvl['speed_min'], lvl['speed_max'])
                new_enemy = Enemy(enemy_img, random.randint(0, 700 - 76), -100, speed)
                enemies.add(new_enemy)

            sprite.groupcollide(bullets, asteroids, True, False)

            if sprite.spritecollide(player, asteroids, False):
                if player_lives > 0:
                    player_lives -= 1
                    player.rect.x = 300
                    player.rect.y = 400
                else:
                    finish = True

            if not finish and sprite.spritecollide(player, enemies, False):
                if player_lives > 0:
                    player_lives -= 1
                    player.rect.x = 300
                    player.rect.y = 400
                else:
                    finish = True

            lvl = levels[current_level]
            if lost >= lvl['lost_limit']:
                finish = True
            elif score >= lvl['score_goal']:
                finish = True
        else:
            if score >= levels[current_level]['score_goal'] and lost < levels[current_level]['lost_limit'] and player_lives >= 0:
                window.blit(win_text, (200, 200))
                if current_level < len(levels) - 1:
                    window.blit(next_level_text, (210, 280))
                    hint = font1.render('Нажмите ENTER для следующего уровня', True, (255, 255, 255))
                    window.blit(hint, (120, 350))
                else:
                    hint = font1.render('Вы прошли все уровни! Нажмите R для рестарта', True, (255, 255, 255))
                    window.blit(hint, (120, 350))
            else:
                window.blit(lose_text, (200, 200))
                hint = font1.render('Нажмите ESC для выхода в меню или R для перезапуска', True, (255, 255, 255))
                window.blit(hint, (120, 350))
            if finish:
                window.blit(restart_text, (230, 320))

    display.update()
    clock.tick(FPS)

quit()
