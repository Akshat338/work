import pygame
from pygame.locals import *
from pygame import mixer
import pickle
import login
import guest
import label
import mysql.connector
import highscore_page
import help_page
from os import path

pygame.mixer.pre_init(44100, -16, 2, 512)
mixer.init()
pygame.init()

clock = pygame.time.Clock()
fps = 60
tile_size = 40
screen_width = tile_size*20
screen_height = tile_size*20

screen = pygame.display.set_mode((screen_width, screen_height))
pygame.display.set_caption('Platformer')


# define font
font = pygame.font.SysFont('Bauhaus 93', 70)
font_score = pygame.font.SysFont('Bauhaus 93', 30)


# define game variables

game_over = 0
main_menu = True
level = 1
max_levels = 7
score = 0


# define colours
white = (255, 255, 255)
blue = (0, 0, 255)


# load images
sun_img = pygame.image.load('img/sun.png')
bg_img = pygame.image.load('img/sky.png')
bg_img = pygame.transform.scale(bg_img, (screen_width, screen_height))

start_img = pygame.image.load('img/start_btn.png')
exit_img = pygame.image.load('img/exit_btn.png')
background_img = pygame.image.load('img1/1.jpg')
background_img = pygame.transform.scale(
    background_img, (screen_width, screen_height))
login_img = pygame.image.load('img1/login.png')
login_img = pygame.transform.scale(login_img, (200, 100))
guest_img = pygame.image.load('img1/guest.png')
guest_img = pygame.transform.scale(guest_img, (200, 100))
help_img = pygame.image.load('img1/help.png')
help_img = pygame.transform.scale(help_img, (200, 100))
play_img = pygame.image.load('img1/play.png')
play_img = pygame.transform.scale(play_img, (150, 150))
exit_img = pygame.image.load('img1/exit1.png')
exit_img = pygame.transform.scale(exit_img, (150, 150))
score_img = pygame.image.load('img1/score.png')
score_img = pygame.transform.scale(score_img, (150, 150))
mainmenu_img = pygame.image.load('img1/mainmenu1.png')
mainmenu_img = pygame.transform.scale(mainmenu_img, (800, 800))
restart_img = pygame.image.load('img1/restart2.png')
restart_img = pygame.transform.scale(restart_img, (150, 150))
mainmenu1_img = pygame.image.load('img1/mainmenu3.png')
mainmenu1_img = pygame.transform.scale(mainmenu1_img, (150, 150))
first_img = pygame.image.load('img1/first.png')
first_img = pygame.transform.scale(first_img, (600, 400))
second_img = pygame.image.load('img1/second.png')
second_img = pygame.transform.scale(second_img, (600, 400))
third_img = pygame.image.load('img1/third.png')
third_img = pygame.transform.scale(third_img, (600, 400))
# load sounds
pygame.mixer.music.load('img/music.wav')
pygame.mixer.music.play(-1, 0.0, 5000)
coin_fx = pygame.mixer.Sound('img/coin.wav')
coin_fx.set_volume(0.5)
jump_fx = pygame.mixer.Sound('img/jump.wav')
jump_fx.set_volume(0.5)
game_over_fx = pygame.mixer.Sound('img/game_over.wav')
game_over_fx.set_volume(0.5)


def draw_text(text, font, text_col, x, y):
    img = font.render(text, True, text_col)
    screen.blit(img, (x, y))


# function to reset level
def reset_level(level):
    player.reset(100, screen_height - 130)
    blob_group.empty()
    platform_group.empty()
    coin_group.empty()
    lava_group.empty()
    exit_group.empty()

    # load in level data and create world
    if path.exists(f'level{level}_data'):
        pickle_in = open(f'level{level}_data', 'rb')
        world_data = pickle.load(pickle_in)
    world = World(world_data)
    # create dummy coin for showing the score

    return world


class Button():
    def __init__(self, x, y, image):
        self.image = image
        self.rect = self.image.get_rect()
        self.rect.x = x
        self.rect.y = y
        self.clicked = False

    def draw(self):
        action = False

        # get mouse position
        pos = pygame.mouse.get_pos()

        # check mouseover and clicked conditions
        if self.rect.collidepoint(pos):
            if pygame.mouse.get_pressed()[0] == 1 and self.clicked == False:
                action = True
                self.clicked = True

        if pygame.mouse.get_pressed()[0] == 0:
            self.clicked = False

        # draw button
        screen.blit(self.image, self.rect)

        return action


class Player():
    def __init__(self, x, y):
        self.reset(x, y)

    def update(self, game_over):
        dx = 0
        dy = 0
        walk_cooldown = 5
        col_thresh = 20

        if game_over == 0:
            # get keypresses
            key = pygame.key.get_pressed()
            if key[pygame.K_SPACE] and self.jumped == False and self.in_air == False:
                jump_fx.play()
                self.vel_y = -15
                self.jumped = True
            if key[pygame.K_SPACE] == False:
                self.jumped = False
            if key[pygame.K_LEFT]:
                dx -= 5
                self.counter += 1
                self.direction = -1
            if key[pygame.K_RIGHT]:
                dx += 5
                self.counter += 1
                self.direction = 1
            if key[pygame.K_LEFT] == False and key[pygame.K_RIGHT] == False:
                self.counter = 0
                self.index = 0
                if self.direction == 1:
                    self.image = self.images_right[self.index]
                if self.direction == -1:
                    self.image = self.images_left[self.index]

            # handle animation
            if self.counter > walk_cooldown:
                self.counter = 0
                self.index += 1
                if self.index >= len(self.images_right):
                    self.index = 0
                if self.direction == 1:
                    self.image = self.images_right[self.index]
                if self.direction == -1:
                    self.image = self.images_left[self.index]

            # add gravity
            self.vel_y += 1
            if self.vel_y > 10:
                self.vel_y = 10
            dy += self.vel_y

            # check for collision
            self.in_air = True
            for tile in world.tile_list:
                # check for collision in x direction
                if tile[1].colliderect(self.rect.x + dx, self.rect.y, self.width, self.height):
                    dx = 0
                # check for collision in y direction
                if tile[1].colliderect(self.rect.x, self.rect.y + dy, self.width, self.height):
                    # check if below the ground i.e. jumping
                    if self.vel_y < 0:
                        dy = tile[1].bottom - self.rect.top
                        self.vel_y = 0
                    # check if above the ground i.e. falling
                    elif self.vel_y >= 0:
                        dy = tile[1].top - self.rect.bottom
                        self.vel_y = 0
                        self.in_air = False

            # check for collision with enemies
            if pygame.sprite.spritecollide(self, blob_group, False):
                game_over = -1
                game_over_fx.play()

            # check for collision with lava
            if pygame.sprite.spritecollide(self, lava_group, False):
                game_over = -1
                game_over_fx.play()

            # check for collision with exit
            if pygame.sprite.spritecollide(self, exit_group, False):
                game_over = 1

            # check for collision with platforms
            for platform in platform_group:
                # collision in the x direction
                if platform.rect.colliderect(self.rect.x + dx, self.rect.y, self.width, self.height):
                    dx = 0
                # collision in the y direction
                if platform.rect.colliderect(self.rect.x, self.rect.y + dy, self.width, self.height):
                    # check if below platform
                    if abs((self.rect.top + dy) - platform.rect.bottom) < col_thresh:
                        self.vel_y = 0

                        dy = platform.rect.bottom - self.rect.top
                    # check if above platform
                    elif abs((self.rect.bottom + dy) - platform.rect.top) < col_thresh:
                        self.rect.bottom = platform.rect.top - 1
                        self.in_air = False
                        dy = 0
                    # move sideways with the platform
                    if platform.move_x != 0:
                        self.rect.x += platform.move_direction

            # update player coordinates
            self.rect.x += dx
            self.rect.y += dy

        elif game_over == -1:
            self.image = self.dead_image

            if self.rect.y > 200:
                self.rect.y -= 5

        # draw player onto screen
        screen.blit(self.image, self.rect)

        return game_over

    def reset(self, x, y):
        self.images_right = []
        self.images_left = []
        self.index = 0
        self.counter = 0
        for num in range(1, 5):
            img_right = pygame.image.load(f'img/guy{num}.png')
            img_right = pygame.transform.scale(img_right, (40, 60))
            img_left = pygame.transform.flip(img_right, True, False)
            self.images_right.append(img_right)
            self.images_left.append(img_left)
        self.dead_image = pygame.image.load('img/ghost.png')
        self.image = self.images_right[self.index]
        self.rect = self.image.get_rect()
        self.rect.x = x
        self.rect.y = y
        self.width = self.image.get_width()
        self.height = self.image.get_height()
        self.vel_y = 0
        self.jumped = False
        self.direction = 0
        self.in_air = True


class World():
    def __init__(self, data):
        self.tile_list = []

        # load images
        dirt_img = pygame.image.load('img/dirt.png')
        grass_img = pygame.image.load('img/grass.png')

        row_count = 0
        for row in data:
            col_count = 0
            for tile in row:
                if tile == 1:
                    img = pygame.transform.scale(
                        dirt_img, (tile_size, tile_size))
                    img_rect = img.get_rect()
                    img_rect.x = col_count * tile_size
                    img_rect.y = row_count * tile_size
                    tile = (img, img_rect)
                    self.tile_list.append(tile)
                if tile == 2:
                    img = pygame.transform.scale(
                        grass_img, (tile_size, tile_size))
                    img_rect = img.get_rect()
                    img_rect.x = col_count * tile_size
                    img_rect.y = row_count * tile_size
                    tile = (img, img_rect)
                    self.tile_list.append(tile)
                if tile == 3:
                    blob = Enemy(col_count * tile_size,
                                 row_count * tile_size + 15)
                    blob_group.add(blob)
                if tile == 4:
                    platform = Platform(
                        col_count * tile_size, row_count * tile_size, 1, 0)
                    platform_group.add(platform)
                if tile == 5:
                    platform = Platform(
                        col_count * tile_size, row_count * tile_size, 0, 1)
                    platform_group.add(platform)
                if tile == 6:
                    lava = Lava(col_count * tile_size, row_count *
                                tile_size + (tile_size // 2))
                    lava_group.add(lava)
                if tile == 7:
                    coin = Coin(col_count * tile_size + (tile_size // 2),
                                row_count * tile_size + (tile_size // 2))
                    coin_group.add(coin)
                if tile == 8:
                    exit = Exit(col_count * tile_size, row_count *
                                tile_size - (tile_size // 2))
                    exit_group.add(exit)
                col_count += 1
            row_count += 1

    def draw(self):
        for tile in self.tile_list:
            screen.blit(tile[0], tile[1])


class Enemy(pygame.sprite.Sprite):
    def __init__(self, x, y):
        pygame.sprite.Sprite.__init__(self)
        self.image = pygame.image.load('img/blob.png')
        self.image = pygame.transform.scale(self.image, (30, 30))
        self.rect = self.image.get_rect()
        self.rect.x = x
        self.rect.y = y
        self.move_direction = 1
        self.move_counter = 0

    def update(self):
        self.rect.x += self.move_direction
        self.move_counter += 1
        if abs(self.move_counter) > 50:
            self.move_direction *= -1
            self.move_counter *= -1


class Platform(pygame.sprite.Sprite):
    def __init__(self, x, y, move_x, move_y):
        pygame.sprite.Sprite.__init__(self)
        img = pygame.image.load('img/platform.png')
        self.image = pygame.transform.scale(img, (tile_size, tile_size // 2))
        self.rect = self.image.get_rect()
        self.rect.x = x
        self.rect.y = y
        self.move_counter = 0
        self.move_direction = 1
        self.move_x = move_x
        self.move_y = move_y

    def update(self):
        self.rect.x += self.move_direction * self.move_x
        self.rect.y += self.move_direction * self.move_y
        self.move_counter += 1
        if abs(self.move_counter) > 50:
            self.move_direction *= -1
            self.move_counter *= -1


class Lava(pygame.sprite.Sprite):
    def __init__(self, x, y):
        pygame.sprite.Sprite.__init__(self)
        img = pygame.image.load('img/lava.png')
        self.image = pygame.transform.scale(img, (tile_size, tile_size // 2))
        self.rect = self.image.get_rect()
        self.rect.x = x
        self.rect.y = y


class Coin(pygame.sprite.Sprite):
    def __init__(self, x, y):
        pygame.sprite.Sprite.__init__(self)
        img = pygame.image.load('img/coin.png')
        self.image = pygame.transform.scale(
            img, (tile_size // 2, tile_size // 2))
        self.rect = self.image.get_rect()
        self.rect.center = (x, y)


class Exit(pygame.sprite.Sprite):
    def __init__(self, x, y):
        pygame.sprite.Sprite.__init__(self)
        img = pygame.image.load('img/exit.png')
        self.image = pygame.transform.scale(
            img, (tile_size, int(tile_size * 1.5)))
        self.rect = self.image.get_rect()
        self.rect.x = x
        self.rect.y = y


player = Player(100, screen_height - 130)

blob_group = pygame.sprite.Group()
platform_group = pygame.sprite.Group()
lava_group = pygame.sprite.Group()
coin_group = pygame.sprite.Group()
exit_group = pygame.sprite.Group()

# create dummy coin for showing the score
score_coin = Coin(tile_size // 2, tile_size // 2)
coin_group.add(score_coin)

# load in level data and create world
if path.exists(f'level{level}_data'):
    pickle_in = open(f'level{level}_data', 'rb')
    world_data = pickle.load(pickle_in)
world = World(world_data)


# create buttons
restart_button = Button(450, 650, restart_img)
start_button = Button(screen_width // 2 - 350, screen_height // 2, start_img)
exit_button = Button(screen_width // 2 + 150, screen_height // 2, exit_img)
login_button = Button(110, 500, login_img)
guest_button = Button(500, 500, guest_img)
help_button = Button(300, 600, help_img)
play_button = Button(150, 500, play_img)
exit_button = Button(550, 500, exit_img)
score_button = Button(350, 500, score_img)
mainmenu1_button = Button(200, 650, mainmenu1_img)

# main loop
run = True
login_check = False
start_game = False
while run:

    clock.tick(fps)

    if login_check == False:
        screen.blit(background_img, (0, 0))
        if login_button.draw():
            login_check, run, record = login.loop(screen)
            for rec in record:
                uid = rec[0]
        if guest_button.draw():
            uid, name, record = guest.create_guest()
            login_check = True
        if help_button.draw():
            help_page.loop(screen)
    else:
        if start_game == False:
            conn = mysql.connector.connect(
                host="localhost", port=3306, user="root", password="", database="shortcut_jump")
            cursor = conn.cursor()
            cursor.execute(f"select * from users where id = {uid}")
            record = cursor.fetchall()
            for rec in record:
                uid = rec[0]
                name = rec[1]
                tot_score = rec[4]
            screen.blit(mainmenu_img, (0, 0))
            label.draw(f'{name}', (0, 0, 0), 250, 210, screen, 70, 'ARIEL')
            label.draw(f'{tot_score}', (0, 0, 0),
                       350, 360, screen, 70, 'ARIEL')
            if play_button.draw():
                start_game = True
                game_over = 0
                world_data = []
                world = reset_level(1)
                score = 0
                level = 1
            if exit_button.draw():
                pygame.time.delay(200)
                login_check = False
            if score_button.draw():
                highscore_page.loop(screen)
        else:
            screen.blit(bg_img, (0, 0))
            screen.blit(sun_img, (100, 100))

            world.draw()

            if game_over == 0:
                blob_group.update()
                platform_group.update()
                # update score
                # check if a coin has been collected
                if pygame.sprite.spritecollide(player, coin_group, True):
                    score += 5
                    coin_fx.play()
                draw_text('X ' + str(score), font_score,
                          white, tile_size - 10, 10)

            blob_group.draw(screen)
            platform_group.draw(screen)
            lava_group.draw(screen)
            coin_group.draw(screen)
            exit_group.draw(screen)

            game_over = player.update(game_over)

            # if player has died
            if game_over == -1:
                conn = mysql.connector.connect(
                    host="localhost", port=3306, user="root", password="", database="shortcut_jump")
                cursor = conn.cursor()
                cursor.execute(
                    "update users set score =%s where id = %s", (score, uid))
                conn.commit()
                cursor.execute("select id,score from users order by score desc")
                record = cursor.fetchall()
                count = 0
                for rec in record:
                    count += 1
                    if rec[0] == uid:
                        rank = count
                        pass
                draw_text(f'Rank:{rank}', font, (255, 255, 255), 480, 350)
                draw_text(f'Score:{score}', font, (255, 255, 255), 60, 350)
                if rank == 1:
                    screen.blit(first_img, (100, 0))
                elif rank == 2:
                    screen.blit(second_img, (100, 0))
                elif rank == 3:
                    screen.blit(third_img, (100, 0))
                else:
                    draw_text('GAME OVER!', font, blue, 200, 100)
                label.draw('TOPSCORES', (255, 0, 0), 280,
                           450, screen, 36, 'Joker Man')
                cursor.execute(
                    "select username,score from users order by score desc limit 3")
                record = cursor.fetchall()
                i = 0
                for rec in record:
                    label.draw(f'{rec[0]}', (255, 0, 0), 250,
                               500+i, screen, 32, 'Joker Man')
                    label.draw(f'{rec[1]}', (255, 0, 0), 500,
                               500+i, screen, 32, 'Joker Man')
                    i += 40
                    pass
                if restart_button.draw():
                    world_data = []
                    world = reset_level(1)
                    game_over = 0
                    score = 0
                    level = 1
                if mainmenu1_button.draw():
                    start_game = False

            # if player has completed the level
            if game_over == 1:
                # reset game and go to next level
                level += 1
                if level <= max_levels:
                    # reset level
                    world_data = []
                    world = reset_level(level)
                    game_over = 0
                else:
                    draw_text('YOU WIN!', font, blue,
                              (screen_width // 2) - 140, screen_height // 2)
                    '''if restart_button.draw():
						level = 1
						#reset level
						world_data = []
						world = reset_level(level)
						game_over = 0
						score = 0
'''
    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            run = False

    pygame.display.update()

pygame.quit()
