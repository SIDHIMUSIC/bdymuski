#!/usr/bin/env python3
"""
Make a Wish — Premium Cinematic Interactive Birthday Animation
Single-file · pygame + numpy · no external assets
"""

import math
import random
import sys
import numpy as np
import pygame

# ──────────────────────────────────────────────
# CONSTANTS & PALETTE
# ──────────────────────────────────────────────
WIDTH, HEIGHT = 800, 1200
FPS = 60
TITLE = "Make a Wish"

IVORY     = (246, 237, 217)
CHAMPAGNE = (236, 208, 140)
GOLD      = (214, 178,  )
NAVY      = (28,  42,  72)
MIDNIGHT  = (8,   12,  28)
SOFT_BLUE = (40,  70, 120)
PINK_GLOW = (255, 200, 220)
WHITE     = (255, 255, 255)
SMOKE     = (180, 180, 200)

SPARK_COLORS = {
    "gold":   (255, 220, 120),
    "white":  (255, 255, 250),
    "rose":   (255, 180, 200),
    "blue":   (160, 200, 255),
    "amber":  (255, 180,  80),
    "champ":  (240, 220, 160),
}

# ──────────────────────────────────────────────
# HELPERS
# ──────────────────────────────────────────────
def clamp(v, lo, hi):
    return max(lo, min(hi, v))

def lerp(a, b, t):
    return a + (b - a) * t

def smooth(t):
    """Smoothstep 0→1"""
    t = clamp(t, 0.0, 1.0)
    return t * t * (3.0 - 2.0 * t)

def ease_out_cubic(t):
    t = clamp(t, 0.0, 1.0)
    return 1.0 - (1.0 - t) ** 3

def ease_out_back(t, s=1.70158):
    t = clamp(t, 0.0, 1.0)
    return 1.0 + (s + 1.0) * (t - 1.0) ** 3 + s * (t - 1.0) ** 2

def approach(current, target, speed, dt):
    """Exponential smoothing toward target."""
    return current + (target - current) * (1.0 - math.exp(-speed * dt))

def scale_col(col, factor):
    return tuple(int(clamp(c * factor, 0, 255)) for c in col)

def cylinder_shade(base, y_norm, light_dir=(-0.6, -0.8)):
    """Simple left-upper lighting for cake body. y_norm 0..1 top→bottom."""
    # Fake cylindrical shading: darker on right, lighter on left
    # We approximate per-row later; this returns a factor
    return 1.0

# ──────────────────────────────────────────────
# PARTICLE SYSTEMS
# ──────────────────────────────────────────────
class Particle:
    __slots__ = ("x", "y", "vx", "vy", "life", "max_life", "size", "color", "gravity", "fade")

    def __init__(self, x, y, vx, vy, life, size, color, gravity=120.0, fade=True):
        self.x = x
        self.y = y
        self.vx = vx
        self.vy = vy
        self.life = life
        self.max_life = life
        self.size = size
        self.color = color
        self.gravity = gravity
        self.fade = fade

    def update(self, dt):
        self.vy += self.gravity * dt
        self.x  += self.vx * dt
        self.y  += self.vy * dt
        self.life -= dt
        return self.life > 0

    def draw(self, surf):
        if self.life <= 0:
            return
        a = self.life / self.max_life if self.fade else 1.0
        r = max(1, int(self.size * a))
        col = scale_col(self.color, a)
        pygame.draw.circle(surf, col, (int(self.x), int(self.y)), r)


class Confetti:
    __slots__ = ("x", "y", "vx", "vy", "rot", "vrot", "w", "h", "color", "life")

    def __init__(self, x, y):
        self.x = x
        self.y = y
        self.vx = random.uniform(-40, 40)
        self.vy = random.uniform(40, 140)
        self.rot = random.uniform(0, 360)
        self.vrot = random.uniform(-180, 180)
        self.w = random.randint(6, 14)
        self.h = random.randint(4, 9)
        self.color = random.choice([
            GOLD, CHAMPAGNE, WHITE, (255, 230, 160),
            (255, 200, 180), (220, 240, 255)
        ])
        self.life = random.uniform(4.0, 7.0)

    def update(self, dt):
        self.vy += 60 * dt
        self.x  += self.vx * dt
        self.y  += self.vy * dt
        self.rot += self.vrot * dt
        self.life -= dt
        return self.life > 0 and self.y < HEIGHT + 40

    def draw(self, surf):
        # Simple rotated rect approximation
        cx, cy = int(self.x), int(self.y)
        points = []
        rad = math.radians(self.rot)
        cosr, sinr = math.cos(rad), math.sin(rad)
        for dx, dy in [(-self.w/2, -self.h/2), (self.w/2, -self.h/2),
                       (self.w/2,  self.h/2), (-self.w/2,  self.h/2)]:
            rx = dx * cosr - dy * sinr
            ry = dx * sinr + dy * cosr
            points.append((cx + rx, cy + ry))
        pygame.draw.polygon(surf, self.color, points)


class Firework:
    def __init__(self, x, y):
        self.particles = []
        n = random.randint(28, 48)
        base_col = random.choice(list(SPARK_COLORS.values()))
        for _ in range(n):
            angle = random.uniform(0, math.tau)
            speed = random.uniform(80, 280)
            vx = math.cos(angle) * speed
            vy = math.sin(angle) * speed
            life = random.uniform(0.7, 1.6)
            size = random.uniform(1.5, 3.5)
            col = random.choice([base_col, WHITE, GOLD, CHAMPAGNE])
            self.particles.append(Particle(x, y, vx, vy, life, size, col, gravity=90))

    def update(self, dt):
        self.particles = [p for p in self.particles if p.update(dt)]
        return len(self.particles) > 0

    def draw(self, surf):
        for p in self.particles:
            p.draw(surf)


# ──────────────────────────────────────────────
# CAKE & CANDLES
# ──────────────────────────────────────────────
class Candle:
    def __init__(self, rel_x, rel_y, stripe="gold"):
        self.rel_x = rel_x      # relative to cake center
        self.rel_y = rel_y
        self.stripe = stripe    # "gold" or "navy"
        self.lit = False
        self.flame_phase = random.uniform(0, math.tau)
        self.flame_h = 0.0
        self.glow = 0.0
        self.smoke = []

    def light(self):
        self.lit = True

    def extinguish(self):
        self.lit = False
        # spawn a few smoke wisps
        for _ in range(4):
            self.smoke.append(Particle(
                0, 0,
                random.uniform(-12, 12), random.uniform(-40, -15),
                random.uniform(0.8, 1.6), random.uniform(3, 7),
                SMOKE, gravity=-20, fade=True
            ))

    def update(self, dt, cake_x, cake_y, hold_boost=0.0):
        self.flame_phase += dt * 9.0
        target_h = 18.0 + hold_boost * 14.0 if self.lit else 0.0
        self.flame_h = approach(self.flame_h, target_h, 8.0, dt)
        target_g = 1.0 + hold_boost * 0.8 if self.lit else 0.0
        self.glow = approach(self.glow, target_g, 6.0, dt)

        # update local smoke
        self.smoke = [s for s in self.smoke if s.update(dt)]
        for s in self.smoke:
            s.x += cake_x + self.rel_x   # temporary absolute for draw later
            # actually keep relative; we offset on draw

    def draw(self, surf, cx, cy, spark_list):
        # candle body
        bw, bh = 10, 38
        bx = int(cx + self.rel_x - bw // 2)
        by = int(cy + self.rel_y - bh)

        if self.stripe == "gold":
            c1, c2 = GOLD, IVORY
        else:
            c1, c2 = NAVY, IVORY

        # striped body
        stripe_h = 5
        for i in range(0, bh, stripe_h):
            col = c1 if (i // stripe_h) % 2 == 0 else c2
            pygame.draw.rect(surf, col, (bx, by + i, bw, min(stripe_h, bh - i)))
        # highlight
        pygame.draw.line(surf, scale_col(IVORY, 1.1), (bx + 2, by + 2), (bx + 2, by + bh - 2), 1)

        # flame
        if self.flame_h > 1.0:
            fx = cx + self.rel_x
            fy = cy + self.rel_y - bh - 4
            h = self.flame_h
            # outer glow (additive-ish)
            for r, a in [(28, 0.08), (18, 0.15), (10, 0.28)]:
                glow_s = pygame.Surface((r * 2, r * 2), pygame.SRCALPHA)
                pygame.draw.circle(glow_s, (*SPARK_COLORS["amber"], int(60 * a * self.glow)), (r, r), r)
                surf.blit(glow_s, (fx - r, fy - r - h * 0.3), special_flags=pygame.BLEND_ADD)

            # flame core – teardrop approximation
            flicker = math.sin(self.flame_phase) * 2.2 + math.sin(self.flame_phase * 2.3) * 1.1
            points = [
                (fx,          fy - h - 2 + flicker * 0.3),
                (fx - 5 - flicker * 0.4, fy - h * 0.35),
                (fx - 3,      fy),
                (fx + 3,      fy),
                (fx + 5 + flicker * 0.4, fy - h * 0.35),
            ]
            pygame.draw.polygon(surf, (255, 230, 120), points)
            # inner bright
            inner = [
                (fx,     fy - h * 0.75 + flicker * 0.2),
                (fx - 2, fy - h * 0.2),
                (fx + 2, fy - h * 0.2),
            ]
            pygame.draw.polygon(surf, (255, 255, 230), inner)

            # occasional spark
            if random.random() < 0.12 * self.glow:
                spark_list.append(Particle(
                    fx + random.uniform(-3, 3), fy - h * 0.6,
                    random.uniform(-25, 25), random.uniform(-80, -30),
                    random.uniform(0.3, 0.7), random.uniform(1.2, 2.4),
                    random.choice([SPARK_COLORS["gold"], SPARK_COLORS["amber"], WHITE]),
                    gravity=30
                ))

        # smoke (relative → absolute)
        for s in self.smoke:
            # smoke particles store relative offsets after update? we handle simply
            px = cx + self.rel_x + s.vx * 0.1   # rough
            # better: smoke keeps its own world position once spawned
            pass  # handled in extinguish + world space below


class Cake:
    def __init__(self):
        self.surface = None
        self.w = 340
        self.h = 280
        self.build()

        # candle layout (relative to cake visual center)
        self.candles = []
        # top tier
        for i, dx in enumerate([-38, 0, 38]):
            stripe = "gold" if i % 2 == 0 else "navy"
            self.candles.append(Candle(dx, -95, stripe))
        # bottom tier
        for i, dx in enumerate([-70, -35, 0, 35, 70]):
            stripe = "navy" if i % 2 == 0 else "gold"
            self.candles.append(Candle(dx, -15, stripe))

        self.y_offset = HEIGHT + 100   # start off-screen
        self.alpha = 0.0
        self.caption_alpha = 0.0

    def build(self):
        """Pre-render the static cake body once."""
        s = pygame.Surface((self.w, self.h), pygame.SRCALPHA)
        cx = self.w // 2

        # plate / stand
        pygame.draw.ellipse(s, (180, 160, 100), (cx - 150, 240, 300, 28))
        pygame.draw.ellipse(s, (210, 190, 130), (cx - 145, 242, 290, 22))
        # soft reflection
        refl = pygame.Surface((280, 40), pygame.SRCALPHA)
        pygame.draw.ellipse(refl, (255, 240, 200, 35), (0, 0, 280, 40))
        s.blit(refl, (cx - 140, 250))

        # bottom tier body (cylinder shading via vertical strips)
        bot_top = 120
        bot_h = 120
        bot_w = 260
        for i in range(bot_w):
            # lighting: brighter on left
            t = i / bot_w
            light = 0.75 + 0.35 * (1.0 - abs(t - 0.35) * 1.6)
            light = clamp(light, 0.55, 1.15)
            col = scale_col(IVORY, light)
            pygame.draw.line(s, col, (cx - bot_w//2 + i, bot_top),
                             (cx - bot_w//2 + i, bot_top + bot_h))
        # bottom frosting scallops
        for i in range(13):
            sx = cx - 125 + i * 20
            pygame.draw.ellipse(s, CHAMPAGNE, (sx - 12, bot_top - 8, 24, 18))
            pygame.draw.ellipse(s, IVORY,     (sx - 9,  bot_top - 5, 18, 13))
        # pearl beads
        for i in range(14):
            px = cx - 125 + i * 19
            pygame.draw.circle(s, (255, 250, 240), (px, bot_top + 4), 3)
            pygame.draw.circle(s, (220, 210, 190), (px, bot_top + 4), 3, 1)

        # navy ribbon
        pygame.draw.rect(s, NAVY, (cx - 128, bot_top + 55, 256, 18))
        for i in range(18):
            dx = cx - 120 + i * 14
            pygame.draw.circle(s, GOLD, (dx, bot_top + 64), 2)

        # top tier
        top_top = 40
        top_h = 85
        top_w = 170
        for i in range(top_w):
            t = i / top_w
            light = 0.78 + 0.32 * (1.0 - abs(t - 0.32) * 1.7)
            light = clamp(light, 0.58, 1.12)
            col = scale_col(IVORY, light)
            pygame.draw.line(s, col, (cx - top_w//2 + i, top_top),
                             (cx - top_w//2 + i, top_top + top_h))
        # top scallops
        for i in range(9):
            sx = cx - 80 + i * 20
            pygame.draw.ellipse(s, CHAMPAGNE, (sx - 11, top_top - 7, 22, 16))
            pygame.draw.ellipse(s, IVORY,     (sx - 8,  top_top - 4, 16, 12))
        # pearls
        for i in range(10):
            px = cx - 80 + i * 18
            pygame.draw.circle(s, (255, 250, 240), (px, top_top + 3), 3)

        self.surface = s

    def update(self, dt, state, hold_t=0.0):
        for c in self.candles:
            c.update(dt, 0, 0, hold_boost=hold_t)

    def draw(self, surf, cx, cy, sparks):
        # cake body
        if self.alpha > 0.01:
            tmp = self.surface.copy()
            tmp.set_alpha(int(255 * self.alpha))
            surf.blit(tmp, (cx - self.w // 2, cy - self.h // 2 + int(self.y_offset)))

        # candles
        for c in self.candles:
            c.draw(surf, cx, cy + int(self.y_offset), sparks)


# ──────────────────────────────────────────────
# SCENE MANAGER
# ──────────────────────────────────────────────
class Scene:
    TITLE = 0
    CAKE_RISE = 1
    CANDLES_LIT = 2
    COUNTDOWN = 3
    WISH = 4
    WISH_MADE = 5
    CELEBRATION = 6


class App:
    def __init__(self):
        pygame.init()
        pygame.display.set_caption(TITLE)
        self.flags = pygame.RESIZABLE
        self.screen = pygame.display.set_mode((WIDTH, HEIGHT), self.flags)
        self.clock = pygame.time.Clock()
        self.fullscreen = False

        # fonts – try elegant serif, fall back gracefully
        self.font_big   = self._font(72)
        self.font_med   = self._font(36)
        self.font_small = self._font(22)
        self.font_tiny  = self._font(16)
        self.font_italic = self._font(26, italic=True)

        self.cake = Cake()
        self.particles = []
        self.confetti = []
        self.fireworks = []
        self.stars = self._make_stars(140)
        self.dust  = self._make_dust(60)

        self.state = Scene.TITLE
        self.t = 0.0          # time in current state
        self.global_t = 0.0
        self.hold_time = 0.0
        self.wish_progress = 0.0
        self.countdown_n = 3
        self.candle_light_idx = 0
        self.caption = ""
        self.caption2 = ""
        self.running = True

        # pre-warm some surfaces
        self.glow_cache = {}

    def _font(self, size, italic=False):
        candidates = [
            "Georgia", "Garamond", "Times New Roman", "Times",
            "Liberation Serif", "DejaVu Serif", "serif"
        ]
        for name in candidates:
            try:
                f = pygame.font.SysFont(name, size, italic=italic)
                if f:
                    return f
            except Exception:
                continue
        return pygame.font.SysFont(None, size, italic=italic)

    def _make_stars(self, n):
        return [(random.randint(0, WIDTH), random.randint(0, HEIGHT),
                 random.uniform(0.3, 1.0), random.uniform(0.5, 2.5))
                for _ in range(n)]

    def _make_dust(self, n):
        return [(random.uniform(0, WIDTH), random.uniform(0, HEIGHT),
                 random.uniform(0.2, 0.7), random.uniform(8, 25))
                for _ in range(n)]

    def reset(self):
        self.cake = Cake()
        self.particles.clear()
        self.confetti.clear()
        self.fireworks.clear()
        self.state = Scene.TITLE
        self.t = 0.0
        self.hold_time = 0.0
        self.wish_progress = 0.0
        self.countdown_n = 3
        self.candle_light_idx = 0
        self.caption = ""
        self.caption2 = ""

    def draw_background(self, surf, intensity=1.0):
        # deep radial gradient
        surf.fill(MIDNIGHT)
        cx, cy = WIDTH // 2, HEIGHT // 2 + 80
        max_r = int(math.hypot(WIDTH, HEIGHT) * 0.55)
        for i in range(12, 0, -1):
            r = int(max_r * i / 12)
            a = int(18 * intensity * (i / 12))
            col = (SOFT_BLUE[0], SOFT_BLUE[1], SOFT_BLUE[2], a)
            g = pygame.Surface((r * 2, r * 2), pygame.SRCALPHA)
            pygame.draw.circle(g, col, (r, r), r)
            surf.blit(g, (cx - r, cy - r))

        # stars
        for x, y, b, s in self.stars:
            twinkle = 0.6 + 0.4 * math.sin(self.global_t * s + x * 0.01)
            a = int(180 * b * twinkle * intensity)
            if a > 10:
                pygame.draw.circle(surf, (200, 220, 255, a), (x, y), 1)

        # floating dust
        for i, (x, y, a, speed) in enumerate(self.dust):
            ny = (y + self.global_t * speed * 3) % (HEIGHT + 20)
            pygame.draw.circle(surf, (220, 230, 255, int(40 * a * intensity)),
                               (int(x), int(ny)), 1)

    def draw_text_gold(self, surf, text, font, y, spacing=4, alpha=255, center=True):
        """Draw gold text with subtle letter-spacing and soft shadow."""
        # render each letter for spacing control
        letters = []
        total_w = 0
        for ch in text:
            g = font.render(ch, True, GOLD)
            letters.append(g)
            total_w += g.get_width() + spacing
        total_w -= spacing

        x = (WIDTH - total_w) // 2 if center else 40
        # shadow
        sx = x
        for g in letters:
            shadow = font.render(text[letters.index(g)] if False else " ", True, (0, 0, 0))
            # simpler: blit dark offset
            dark = g.copy()
            dark.fill((0, 0, 0, 90), special_flags=pygame.BLEND_RGBA_MULT)
            surf.blit(dark, (sx + 2, y + 2))
            sx += g.get_width() + spacing

        # main
        sx = x
        for g in letters:
            if alpha < 255:
                g = g.copy()
                g.set_alpha(alpha)
            surf.blit(g, (sx, y))
            sx += g.get_width() + spacing

    def draw_text_simple(self, surf, text, font, y, color=IVORY, alpha=255, italic=False):
        f = self.font_italic if italic else font
        g = f.render(text, True, color)
        if alpha < 255:
            g = g.copy()
            g.set_alpha(alpha)
        surf.blit(g, ((WIDTH - g.get_width()) // 2, y))

    def spawn_firework(self):
        x = random.randint(80, WIDTH - 80)
        y = random.randint(80, HEIGHT // 2 - 40)
        self.fireworks.append(Firework(x, y))

    def update(self, dt):
        self.global_t += dt
        self.t += dt

        keys = pygame.key.get_pressed()

        # particles always
        self.particles = [p for p in self.particles if p.update(dt)]
        self.confetti  = [c for c in self.confetti  if c.update(dt)]
        self.fireworks = [f for f in self.fireworks if f.update(dt)]

        # ── STATE MACHINE ──
        if self.state == Scene.TITLE:
            if keys[pygame.K_RETURN] or keys[pygame.K_KP_ENTER]:
                self.state = Scene.CAKE_RISE
                self.t = 0.0

        elif self.state == Scene.CAKE_RISE:
            # rise & fade in
            p = ease_out_cubic(min(1.0, self.t / 2.8))
            self.cake.y_offset = lerp(220, 0, p)
            self.cake.alpha = smooth(min(1.0, self.t / 1.8))
            self.cake.caption_alpha = smooth(clamp((self.t - 1.2) / 1.2, 0, 1))
            self.caption = "Someone special is celebrating today..."
            if self.t > 3.6:
                self.state = Scene.CANDLES_LIT
                self.t = 0.0
                self.candle_light_idx = 0

        elif self.state == Scene.CANDLES_LIT:
            # light candles one by one
            interval = 0.38
            while self.candle_light_idx < len(self.cake.candles) and \
                  self.t > self.candle_light_idx * interval:
                self.cake.candles[self.candle_light_idx].light()
                self.candle_light_idx += 1
            self.caption = "Before we celebrate..."
            self.caption2 = "Close your eyes and make a wish."
            if self.t > len(self.cake.candles) * interval + 1.8:
                self.state = Scene.COUNTDOWN
                self.t = 0.0
                self.countdown_n = 3

        elif self.state == Scene.COUNTDOWN:
            self.countdown_n = 3 - int(self.t)
            if self.t > 3.2:
                self.state = Scene.WISH
                self.t = 0.0
                self.wish_progress = 0.0
                self.hold_time = 0.0

        elif self.state == Scene.WISH:
            holding = keys[pygame.K_SPACE]
            if holding:
                self.hold_time += dt
                self.wish_progress = min(1.0, self.wish_progress + dt * 0.55)
            else:
                self.wish_progress = max(0.0, self.wish_progress - dt * 0.35)

            self.cake.update(dt, self.state, hold_t=self.wish_progress)

            if self.wish_progress >= 1.0:
                # blow out
                for c in self.cake.candles:
                    if c.lit:
                        c.extinguish()
                        # world-space smoke
                        for _ in range(5):
                            self.particles.append(Particle(
                                WIDTH//2 + c.rel_x + random.uniform(-6, 6),
                                HEIGHT//2 - 40 + c.rel_y - 40,
                                random.uniform(-20, 20), random.uniform(-55, -15),
                                random.uniform(1.0, 2.0), random.uniform(4, 9),
                                SMOKE, gravity=-25
                            ))
                self.state = Scene.WISH_MADE
                self.t = 0.0

        elif self.state == Scene.WISH_MADE:
            if self.t > 3.2:
                self.state = Scene.CELEBRATION
                self.t = 0.0

        elif self.state == Scene.CELEBRATION:
            # continuous confetti + fireworks
            if random.random() < 0.07:
                self.confetti.append(Confetti(random.randint(0, WIDTH), -20))
            if random.random() < 0.025 and len(self.fireworks) < 5:
                self.spawn_firework()
            # keep some ambient sparks
            if random.random() < 0.04:
                self.particles.append(Particle(
                    random.randint(100, WIDTH-100), random.randint(60, 400),
                    random.uniform(-30, 30), random.uniform(-40, 10),
                    random.uniform(0.4, 1.0), random.uniform(1.5, 3),
                    random.choice(list(SPARK_COLORS.values())), gravity=40
                ))

        # always update cake candles for flame animation
        if self.state >= Scene.CANDLES_LIT:
            boost = self.wish_progress if self.state == Scene.WISH else 0.0
            self.cake.update(dt, self.state, hold_t=boost)

    def draw(self):
        # base
        intensity = 1.0
        if self.state == Scene.WISH:
            intensity = lerp(1.0, 0.45, self.wish_progress * 0.7)
        elif self.state == Scene.WISH_MADE:
            intensity = 0.55
        elif self.state == Scene.CELEBRATION:
            intensity = 0.85

        self.draw_background(self.screen, intensity)

        cx, cy = WIDTH // 2, HEIGHT // 2 + 40

        # pink glow bloom behind cake in later scenes
        if self.state >= Scene.WISH_MADE:
            bloom = min(1.0, self.t * 0.6) if self.state == Scene.WISH_MADE else 1.0
            for r, a in [(220, 18), (140, 28), (80, 40)]:
                g = pygame.Surface((r*2, r*2), pygame.SRCALPHA)
                pygame.draw.circle(g, (*PINK_GLOW, int(a * bloom)), (r, r), r)
                self.screen.blit(g, (cx - r, cy - r + 30), special_flags=pygame.BLEND_ADD)

        # spotlight in celebration
        if self.state == Scene.CELEBRATION:
            # soft vertical beam
            beam = pygame.Surface((320, HEIGHT), pygame.SRCALPHA)
            for i in range(20):
                a = int(12 * (1.0 - i / 20))
                pygame.draw.rect(beam, (255, 245, 220, a),
                                 (160 - i*6, 0, 12 + i*12, HEIGHT))
            self.screen.blit(beam, (cx - 160, 0), special_flags=pygame.BLEND_ADD)

        # cake
        if self.state >= Scene.CAKE_RISE:
            self.cake.draw(self.screen, cx, cy, self.particles)

        # particles / confetti / fireworks
        for p in self.particles:
            p.draw(self.screen)
        for c in self.confetti:
            c.draw(self.screen)
        for f in self.fireworks:
            f.draw(self.screen)

        # ── UI TEXT ──
        if self.state == Scene.TITLE:
            self.draw_text_gold(self.screen, "MAKE A WISH", self.font_big, 340, spacing=10)
            self.draw_text_simple(self.screen, "an interactive birthday surprise",
                                  self.font_italic, 430, color=CHAMPAGNE, italic=True)
            blink = 0.5 + 0.5 * math.sin(self.global_t * 2.8)
            self.draw_text_simple(self.screen, "Press ENTER to begin",
                                  self.font_small, 720, color=IVORY,
                                  alpha=int(180 * blink))

        elif self.state == Scene.CAKE_RISE:
            a = int(255 * self.cake.caption_alpha)
            self.draw_text_simple(self.screen, self.caption, self.font_small, 160,
                                  color=CHAMPAGNE, alpha=a)

        elif self.state == Scene.CANDLES_LIT:
            self.draw_text_simple(self.screen, self.caption, self.font_small, 140,
                                  color=CHAMPAGNE)
            a = int(200 * smooth(clamp((self.t - 1.0) / 1.2, 0, 1)))
            self.draw_text_simple(self.screen, self.caption2, self.font_italic, 175,
                                  color=IVORY, alpha=a, italic=True)

        elif self.state == Scene.COUNTDOWN:
            n = max(1, self.countdown_n)
            # big number with scale pulse
            pulse = 1.0 + 0.08 * math.sin(self.t * 8)
            # we approximate by font size choice
            self.draw_text_gold(self.screen, str(n), self.font_big, 200, spacing=2)

        elif self.state == Scene.WISH:
            self.draw_text_simple(self.screen, "Press and hold SPACE to make your wish.",
                                  self.font_small, 130, color=IVORY)
            # progress bar
            bw, bh = 280, 4
            bx = (WIDTH - bw) // 2
            by = 175
            pygame.draw.rect(self.screen, (60, 60, 80), (bx, by, bw, bh), border_radius=2)
            fw = int(bw * self.wish_progress)
            if fw > 0:
                pygame.draw.rect(self.screen, GOLD, (bx, by, fw, bh), border_radius=2)
                # soft glow on bar
                g = pygame.Surface((fw + 20, bh + 16), pygame.SRCALPHA)
                pygame.draw.rect(g, (*GOLD, 40), (10, 6, fw, bh), border_radius=2)
                self.screen.blit(g, (bx - 10, by - 6), special_flags=pygame.BLEND_ADD)

        elif self.state == Scene.WISH_MADE:
            a = int(255 * smooth(min(1.0, self.t / 0.8)))
            self.draw_text_gold(self.screen, "WISH MADE.", self.font_big, 200, spacing=8)
            self.draw_text_simple(self.screen, "Now let the beautiful things begin.",
                                  self.font_italic, 290, color=CHAMPAGNE,
                                  alpha=a, italic=True)

        elif self.state == Scene.CELEBRATION:
            self.draw_text_gold(self.screen, "HAPPY BIRTHDAY!", self.font_big, 160, spacing=6)
            self.draw_text_simple(self.screen, "May every little wish find its way to you.",
                                  self.font_italic, 250, color=CHAMPAGNE, italic=True)
            blink = 0.45 + 0.55 * math.sin(self.global_t * 2.2)
            self.draw_text_simple(self.screen, "Press R to replay",
                                  self.font_tiny, 1100, color=IVORY,
                                  alpha=int(160 * blink))

        # always show tiny controls
        hint = self.font_tiny.render("R restart  ·  F fullscreen  ·  ESC quit", True, (120, 130, 150))
        hint.set_alpha(90)
        self.screen.blit(hint, ((WIDTH - hint.get_width()) // 2, HEIGHT - 28))

        pygame.display.flip()

    def run(self):
        while self.running:
            dt = min(self.clock.tick(FPS) / 1000.0, 0.05)

            for event in pygame.event.get():
                if event.type == pygame.QUIT:
                    self.running = False
                elif event.type == pygame.KEYDOWN:
                    if event.key == pygame.K_ESCAPE:
                        self.running = False
                    elif event.key == pygame.K_r:
                        self.reset()
                    elif event.key == pygame.K_f:
                        self.fullscreen = not self.fullscreen
                        if self.fullscreen:
                            self.screen = pygame.display.set_mode((0, 0), pygame.FULLSCREEN)
                        else:
                            self.screen = pygame.display.set_mode((WIDTH, HEIGHT), self.flags)

            self.update(dt)
            self.draw()

        pygame.quit()
        sys.exit(0)


# ──────────────────────────────────────────────
if __name__ == "__main__":
    App().run()
