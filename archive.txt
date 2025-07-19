car = {
  x = 18*8, y = 6*8,
  dx = 0, dy = 0,
  angle = 0.5,
  boost = 10,
  speed = 0,
  frozen = false,

  -- dash / flick
  last_flick_time = -10,
  flick_active = false,
  flick_timer = 0,
  sprite_id = 0,
  flick_cooldown = 0,

  -- animaciones
  boost_phase = nil,
  boost_anim_timer = 0,
  boost_start_timer = 0,
  grass_anim_frame = 0,
  last_grass_x = 0,
  last_grass_y = 0,

  update = function(self)
    if self.frozen then return end

    if btn(0) then self.angle += 0.05 end
    if btn(1) then self.angle -= 0.05 end

    if btn(2) then
      self.dx += sin(self.angle)*0.1
      self.dy += -cos(self.angle)*0.1
    end

    if btn(3) then
      self.dx -= sin(self.angle)*0.05
      self.dy -= -cos(self.angle)*0.05
    end

    if btn(4) and self.boost > 0 then
      self.dx += sin(self.angle)*0.2
      self.dy += -cos(self.angle)*0.2
      self.boost -= 0.2
    end

    if self.flick_cooldown > 0 then
      self.flick_cooldown -= 1/30
    end

    if btnp(5) and self.flick_cooldown <= 0 then
      if time() - self.last_flick_time < 0.25 then
        local dash_strength = 2.0
        local dash_angle = self.angle
        if btn(3) then dash_angle += 0.5 end

        self.dx += sin(dash_angle) * dash_strength
        self.dy += -cos(dash_angle) * dash_strength

        self.flick_active = true
        self.flick_timer = 6
        self.sprite_id = 10
        self.flick_cooldown = 0.5
      end
      self.last_flick_time = time()
    end

    if self.flick_active then
      self.flick_timer -= 1
      if self.flick_timer <= 0 then
        self.flick_active = false
        self.sprite_id = 0
      end
    end

    local max_speed = 4.5
    local current_speed = sqrt(self.dx^2 + self.dy^2)
    if current_speed > max_speed then
      local scale = max_speed / current_speed
      self.dx *= scale
      self.dy *= scale
    end

    self.dx *= 0.93
    self.dy *= 0.93

    self.x += self.dx
    self.y += self.dy

    local min_x, max_x = 0, 39*8
    local min_y, max_y = 2*8, 25*8

    if self.x < min_x then self.x = min_x self.dx *= -0.5 end
    if self.x > max_x - 16 then self.x = max_x - 16 self.dx *= -0.5 end
    if self.y < min_y then self.y = min_y self.dy *= -0.5 end
    if self.y > max_y - 16 then self.y = max_y - 16 self.dy *= -0.5 end

    self.speed = sqrt(self.dx^2 + self.dy^2)

    local moved = abs(self.x - self.last_grass_x) + abs(self.y - self.last_grass_y)
    if moved > 4 then
      self.grass_anim_frame = 1 - self.grass_anim_frame
      self.last_grass_x = self.x
      self.last_grass_y = self.y
    end
  end,

  draw = function(self)
    local cx, cy = self.x + 8, self.y + 8
    local bx = cx + sin(self.angle) * -15
    local by = cy + -cos(self.angle) * -15

    draw_boost_animation(self, bx, by)

    if self.speed > 0.1 and not btn(4) then
      local grass_sprite = (self.grass_anim_frame == 0) and 224 or 226
      draw_rotated_sprite(grass_sprite, bx, by, 16, 16, self.angle)
    end

    draw_rotated_sprite(self.sprite_id, cx, cy, 16, 16, self.angle)
    draw_arrow(self)
  end
}

function draw_boost_animation(self, bx, by)
  if not btn(4) then
    self.boost_phase = nil
    self.boost_start_timer = 0
    self.boost_anim_timer = 0
    return
  end

  if self.boost <= 0 then
    draw_rotated_sprite(204, bx, by, 16, 16, self.angle)
    return
  end

  local seq_start = {192, 194, 196}
  local seq_loop = {198, 200, 202, 200}
  local sprite = -1

  if self.boost_phase == nil then
    self.boost_phase = "start"
    self.boost_start_timer = 0
    self.boost_anim_timer = 0
  end

  if self.boost_phase == "start" then
    local i = self.boost_start_timer \ 4
    if i < #seq_start then
      sprite = seq_start[i + 1]
      self.boost_start_timer += 1
    else
      self.boost_phase = "loop"
      self.boost_anim_timer = 0
    end
  end

  if self.boost_phase == "loop" then
    local i = (self.boost_anim_timer \ 4) % #seq_loop
    sprite = seq_loop[i + 1]
    self.boost_anim_timer += 1
  end

  if sprite >= 0 then
    draw_rotated_sprite(sprite, bx, by, 16, 16, self.angle)
  end
end

function draw_arrow(self)
  local dx = ball.x + 8 - (self.x + 8)
  local dy = ball.y + 8 - (self.y + 8)
  local angle = atan2(dy, dx)
  local dist = sqrt(dx*dx + dy*dy)
  local fx = self.x + 8 + (dx / dist) * 12
  local fy = self.y + 8 + (dy / dist) * 12
  draw_rotated_sprite(42, fx - 4, fy - 4, 8, 8, angle)
end

function draw_rotated_sprite(spr_id, cx, cy, w, h, angle)
  local sx = (spr_id % 16) * 8
  local sy = flr(spr_id / 16) * 8
  for x = 0, w - 1 do
    for y = 0, h - 1 do
      local px, py = sx + x, sy + y
      local c = sget(px, py)
      if c ~= 15 then
        local dx = x - w / 2
        local dy = y - h / 2
        local rx = cos(angle) * dx - sin(angle) * dy
        local ry = sin(angle) * dx + cos(angle) * dy
        pset(cx + rx, cy + ry, c)
      end
    end
  end
end


-- configuraciれはn mejorada de ias
function create_ia(team, x, y, sprite)
    local ia = {
        sprite = sprite or 6,
        x = x,
        y = y,
        dx = 0,
        dy = 0,
        angle = 0.5,
        turn_rate = 0.05,
        team = team,
        boost = 4,
        freeze_start = 0,
        target_x = x,
        target_y = y,
        push_force = 1.2,  -- fuerza mれくs moderada
        detection_radius = 80,
        speed = 0.15,      -- velocidad reducida
        avoidance_force = 0.2  -- nueva: fuerza para evitar agrupamiento
    }
    return ia
end

-- funciれはn de actualizaciれはn mejorada
function update_ia(ia)
    if ia.freeze_start > 0 then
        ia.freeze_start -= 1
        return
    end

    local cx, cy = ia.x + 8, ia.y + 8
    local bx, by = ball.x + 4, ball.y + 4

    -- 1. comportamiento estratれたgico por equipo
    local target_x, target_y = bx, by  -- por defecto sigue la pelota
    
    if ia.team == "blue" then
        -- comportamiento defensivo
        local ball_to_blue_goal = sqrt((bx-2*8)^2 + (by-13*8)^2)
        if ball_to_blue_goal < 120 then
            target_x, target_y = (bx + 2*8)/2, (by + 13*8)/2  -- posiciれはn intermedia
        end
    else
        -- comportamiento ofensivo
        local ball_to_orange_goal = sqrt((bx-37*8)^2 + (by-13*8)^2)
        if ball_to_orange_goal > 100 then
            target_x, target_y = (bx + 37*8)/2, (by + 13*8)/2
        end
    end

    -- 2. movimiento suavizado hacia el objetivo
    local dx = target_x - cx
    local dy = target_y - cy
    local dist = sqrt(dx*dx + dy*dy)
    
    if dist > 5 then  -- solo mover si estれく suficientemente lejos
        local target_angle = atan2(dy, dx)
        local angle_diff = ((target_angle - ia.angle + 0.5) % 1) - 0.5
        
        ia.angle = (ia.angle + mid(-ia.turn_rate, angle_diff, ia.turn_rate)) % 1
        
        if abs(angle_diff) < 0.25 then  -- solo acelerar si estれく alineado
            local force = min(ia.speed, dist/100)  -- fuerza proporcional a la distancia
            ia.dx += sin(ia.angle) * force
            ia.dy += -cos(ia.angle) * force
        end
    end

    -- 3. fれとsica mejorada
    ia.dx *= 0.95  -- mayor fricciれはn
    ia.dy *= 0.95
    ia.x += ia.dx
    ia.y += ia.dy

    -- 4. lれとmites del campo con rebote mejorado
    local min_x, max_x = 8, 38*8
    local min_y, max_y = 2*8, 24*8
    
    if ia.x < min_x then 
        ia.x = min_x 
        ia.dx = abs(ia.dx) * 0.7  -- rebote mれくs suave
    end
    if ia.x > max_x then 
        ia.x = max_x 
        ia.dx = -abs(ia.dx) * 0.7
    end
    if ia.y < min_y then 
        ia.y = min_y 
        ia.dy = abs(ia.dy) * 0.7
    end
    if ia.y > max_y then 
        ia.y = max_y 
        ia.dy = -abs(ia.dy) * 0.7
    end

    -- 5. interacciれはn con pelota (mれくs controlada)
    local ball_dist = sqrt((bx-cx)^2 + (by-cy)^2)
    if ball_dist < 14 then
        local push_angle = atan2(by-cy, bx-cx)
        ball.dx += cos(push_angle) * ia.push_force * min(1, (14-ball_dist)/4)
        ball.dy += sin(push_angle) * ia.push_force * min(1, (14-ball_dist)/4)
    end

    -- 6. evitaciれはn de colisiones mejorada
    local avoidance_x, avoidance_y = 0, 0
    
    -- con el jugador
    local player_dist = sqrt((ia.x-car.x)^2 + (ia.y-car.y)^2)
    if player_dist < 24 then
        local avoid_angle = atan2(ia.y-car.y, ia.x-car.x)
        avoidance_x += cos(avoid_angle) * ia.avoidance_force
        avoidance_y += sin(avoid_angle) * ia.avoidance_force
    end
    
    -- con otras ias
    for other in all(ias) do
        if other ~= ia then
            local other_dist = sqrt((ia.x-other.x)^2 + (ia.y-other.y)^2)
            if other_dist < 24 then
                local avoid_angle = atan2(ia.y-other.y, ia.x-other.x)
                avoidance_x += cos(avoid_angle) * ia.avoidance_force * 0.6
                avoidance_y += sin(avoid_angle) * ia.avoidance_force * 0.6
            end
        end
    end
    
    -- aplicar fuerzas de evitaciれはn
    ia.dx += avoidance_x
    ia.dy += avoidance_y
end





ball = {
  x = 120, y = 120,
  dx = 0, dy = 0,

  update = function(self)
    if game.freeze then return end
    self.x += self.dx
    self.y += self.dy

    local min_x, max_x = 0, 39*8
    local min_y, max_y = 2*8, 25*8

    local speed = sqrt(self.dx^2 + self.dy^2)

    if self.y <= min_y and self.x >= 16*8 and self.x <= 21*8 then
      game:goal("blue", speed) return
    elseif self.y >= max_y - 16 and self.x >= 16*8 and self.x <= 21*8 then
      game:goal("orange", speed) return
    end

    if self.x < min_x then self.x = min_x self.dx *= -1 end
    if self.x > max_x - 16 then self.x = max_x - 16 self.dx *= -1 end
    if self.y < min_y then self.y = min_y self.dy *= -1 end
    if self.y > max_y - 16 then self.y = max_y - 16 self.dy *= -1 end

    self.dx *= 0.98
    self.dy *= 0.98

    local d = sqrt((self.x - car.x)^2 + (self.y - car.y)^2)
    if d < 16 then
      self.dx = (self.x - car.x) * 0.2 + car.dx * 0.3
      self.dy = (self.y - car.y) * 0.2 + car.dy * 0.3
    end
  end,

  draw = function(self)
    spr(8, self.x, self.y, 2, 2)
  end,

  reset = function(self)
    self.x = 120
    self.y = 120
    self.dx = 0
    self.dy = 0
  end
}



-- pads de boost
boost_pads = {
  pads = {
    {x=3*8, y=21*8, active=true},
    {x=3*8, y=5*8, active=true},
    {x=33*8, y=21*8, active=true},
    {x=33*8, y=5*8, active=true}
  },

  draw = function(self)
    for p in all(self.pads) do
      spr(p.active and 128 or 130, p.x, p.y, 2, 2)
    end
  end,

  check = function(self)
    for p in all(self.pads) do
      if p.active and abs(car.x - p.x) < 12 and abs(car.y - p.y) < 12 then
        car.boost = 10
        p.active = false
        p.cooldown = 300
      end
      if p.cooldown then
        p.cooldown -= 1
        if p.cooldown <= 0 then
          p.active = true
          p.cooldown = nil
        end
      end
    end
  end
}



function extract_speed(msg)
  local spd = ""
  local in_paren = false
  for i=1,#msg do
    local ch = sub(msg, i, i)
    if ch == "(" then
      in_paren = true
    elseif ch == ")" then
      break
    elseif in_paren then
      spd = spd .. ch
    end
  end
  return spd
end

function str_has(s, word)
  for i=1,#s-#word+1 do
    if sub(s, i, i+#word-1) == word then
      return true
    end
  end
  return false
end

game = {
  timer = 60 * 3 * 30,
  score_blue = 0,
  score_orange = 0,
  freeze = false,
  freeze_timer = 0,
  goal_message = "",
  countdown_timer = 30,
  mode = "1v1",
  state = "menu",

  goal = function(self, team, speed)
    self.freeze = true
    self.freeze_timer = 45
    local spd = speed and flr(speed * 10) or "?"
    self.goal_message = "gol de " .. team .. " (" .. spd .. ")"

    if team == "blue" then
      self.score_blue += 1
    else
      self.score_orange += 1
    end

    if ball and ball.reset then ball:reset() end
    if car then
      car.x = 18 * 8
      car.y = 6 * 8
      car.dx = 0
      car.dy = 0
      car.angle = 0.5
      car.boost = 4
    end

    if iatest then
      if team == "blue" then
        iatest.x = 28 * 8
        iatest.y = 13 * 8
        iatest.angle = 0.5
      else
        iatest.x = 10 * 8
        iatest.y = 13 * 8
        iatest.angle = 0
      end
      iatest.dx = 0
      iatest.dy = 0
      iatest.boost = 4
      if ball then
        iatest.target_x = ball.x + 4
        iatest.target_y = ball.y + 4
      end
    end
  end,

  update = function(self)
    if self.freeze then
      self.freeze_timer -= 1
      if self.freeze_timer <= 0 then
        self.freeze = false
        self.goal_message = ""
        self.countdown_timer = 30
      end
      return
    end

    if self.countdown_timer > 0 then
      self.countdown_timer -= 1
      return
    end

    if self.timer == 60 * 3 * 30 and self.countdown_timer == 0 then
      self.countdown_timer = 30
    end

    if self.timer > 0 then
      self.timer -= 1
    elseif self.timer == 0 then
      if not self.freeze and self.score_blue != self.score_orange then
        self.goal_message = self.score_blue > self.score_orange and "gana azul" or "gana naranja"
        self.freeze = true
        self.freeze_timer = 300
      elseif not self.freeze then
        self.goal_message = "empate!"
        self.freeze = true
        self.freeze_timer = 300
      end
    end

    if car and car.update then car:update() end
    if update_iatest then update_iatest() end
    if ball and ball.update then ball:update() end
    if boost_pads and boost_pads.check then boost_pads:check() end

    if iatest and car then
      local ia = iatest
      local dist_x = (ia.x + 8) - (car.x + 8)
      local dist_y = (ia.y + 8) - (car.y + 8)
      local cd_sq = dist_x * dist_x + dist_y * dist_y
      if cd_sq < 16 * 16 then
        local actual_dist = sqrt(cd_sq)
        if actual_dist == 0 then actual_dist = 0.001 end
        local angle_to_car = atan2(dist_y, dist_x)
        local rad = angle_to_car
        local push_force = 0.5
        ia.dx -= cos(rad) * push_force
        ia.dy -= sin(rad) * push_force
        car.dx += cos(rad) * push_force
        car.dy += sin(rad) * push_force
      end
    end
  end,

  draw = function(self)
    if car then
      camera(car.x - 64 + 8, car.y - 64 + 8)
    else
      camera()
    end

    cls(0)
    map(0, 0, 0, 0, 64, 64)

    if car and car.draw then car:draw() end
    if draw_iatest_global then draw_iatest_global() end
    if ball and ball.draw then ball:draw() end
    if boost_pads and boost_pads.draw then boost_pads:draw() end

    camera()

    local time_left = flr(self.timer / 30)
    local s_orange = tostr(self.score_orange)
    local s_blue = tostr(self.score_blue)
    local s_time = tostr(time_left)

    local bw = 16
    local bh = 10
    local total_w = 3 * bw
    local base_x = 64 - total_w / 2
    local y = 3

    rectfill(base_x, y, base_x + bw - 1, y + bh, 9)
    print(s_orange, base_x + 5, y + 2, 7)

    rectfill(base_x + bw, y, base_x + 2 * bw - 1, y + bh, 5)
    print(s_time, base_x + bw + 4, y + 2, 7)

    rectfill(base_x + 2 * bw, y, base_x + 3 * bw - 1, y + bh, 12)
    print(s_blue, base_x + 2 * bw + 5, y + 2, 7)

    -- boost bar
    if car then
      local max_boost = 10
      local boost = car.boost or 0
      local pct = min(boost / max_boost, 1)
      local bar_w = 30
      local bar_h = 5
      local fill = flr(pct * bar_w)
      local x = 127 - bar_w - 4
      local yb = 127 - bar_h - 4

      rectfill(x, yb, x + bar_w, yb + bar_h, 0)
      rectfill(x, yb, x + fill, yb + bar_h, 10)
      rect(x, yb, x + bar_w, yb + bar_h, 7)
    end

    -- cuenta regresiva grande centrada
    if self.countdown_timer > 0 then
      local count = ceil(self.countdown_timer / 10)
      local text = tostring(count)
      local w = #text * 8
      local x = 64 - w / 2
      local y = 50
      print(text, x + 1, y + 1, 0)
      print(text, x, y, 7)
      print(text, x, y + 1, 7)
      print(text, x + 1, y, 7)
    end

    -- cartel de gol
    if self.freeze and self.goal_message ~= "" then
      local msg = self.goal_message
      local speed_txt = extract_speed(msg)

      local msg1 = msg
      local msg2 = "velocidad: " .. speed_txt .. " km/h"

      local w = max(#msg1, #msg2) * 4 + 20
      local x = (128 - w) / 2
      local y = 52
      local h = 25

      local bg_color = 1
      if str_has(msg, "blue") then bg_color = 12 end
      if str_has(msg, "orange") then bg_color = 9 end
      if str_has(msg, "empate") then bg_color = 5 end

      rectfill(x, y, x + w, y + h, bg_color)
      print(msg1, x + 10, y + 4, 7)
      print(msg2, x + 10, y + 14, 7)
    end
  end
}




function _init()
  palt(15, true)
  game.state = "menu"
end

function _update()
  if game.state == "menu" then
    menu:update()
  else
    game:update()
  end
end

function _draw()
  if game.state == "menu" then
    menu:draw()
  else
    game:draw()
  end
end


--menu

menu = {
  options = {"1v1", "2v2", "prれくctica"},
  selected = 1,

  update = function(self)
    if btnp(2) then
      self.selected = (self.selected - 2) % #self.options + 1
    end
    if btnp(3) then
      self.selected = (self.selected % #self.options) + 1
    end
    if btnp(4) or btnp(5) then
      local opt = self.options[self.selected]
      if opt == "prれくctica" then
        game.mode = "practica"
      elseif opt == "1v1" then
        game.mode = "1v1"
      elseif opt == "2v2" then
        game.mode = "2v2"
      end
      game.timer = 60 * 3 * 30
      game.countdown_timer = 30
      game.freeze = false
      game.goal_message = ""
      game.state = "game"
    end
  end,

  draw = function(self)
    cls(0)
    map(0, 0, 0, 0, 64, 64)
    print("ヌあや rocket car ヌあや", 32, 20, 7)
    print("elige modo:", 42, 36, 6)

    for i, opt in ipairs(self.options) do
      local y = 48 + i * 10
      local txt = (i == self.selected and "ヌおさ " or "  ") .. opt
      print(txt, 40, y, i == self.selected and 11 or 5)
    end

    print("ヌて●ヌて♥ para mover, ❎ para elegir", 15, 110, 6)
  end
}
