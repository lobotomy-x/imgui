# Pong — Original Lua Script (UEVR)

This is the original Pong game written in Lua for the
[UEVR](https://github.com/praydog/UEVR) (Unreal Engine VR) framework,
before it was rewritten in C++ as a Dear ImGui demo example.

The script is preserved here unmodified as the starting point for the
C++ port found in `imgui_demo.cpp` → `ShowExampleAppPong()`.

---

```lua
local pong = {}
local vr = uevr.params.vr
local api = uevr.api

local pong = {}

-- this is effecient but suboptimal, don't do it outside of toys like this
local v2f = function(t)
    return Vector2f.new(table.unpack(t))
end

local floor, max, min, random, cos, sin, pi = math.floor, math.max, math.min, math.random, math.cos, math.sin, math.pi


function VecToU32(vec)
    local r = floor(max(0, min(255, vec.x * 255 + 0.5)))
    local g = floor(max(0, min(255, vec.y * 255 + 0.5)))
    local b = floor(max(0, min(255, vec.z * 255 + 0.5)))
    local a = floor(max(0, min(255, vec.w * 255 + 0.5)))

    -- ImGui expects 0xAABBGGRR
    return (a << 24) | (b << 16) | (g << 8) | r
end

local v4f = function(t)
    return Vector4f.new(table.unpack(t))
end
local left_stick

local res = imgui.get_display_size() or v2f{1920,1080}
local playercolor = 0xFFDBDDDE
local playerh = 120
local playerw = 60

local ball_r = 24
local ball_segs = 18

local ballpos = res * 0.5
local enemyy = 0.5 * res.y
local miny = 0
local Score = {0,0}
local playery = 0.5 * res.y
local dt = 0.00167
local enemy_target_y = 0.5 * res.y
local enemy_speed = 1000 -- Slightly slower than player to make it beatable
local ai_precision = 0.9 -- 1.0 is perfect, lower is more "human"

local move_speed = 1200
local is_resetting = true

local fps = 60
local function draw_ball(pos)
    draw.filled_circle(pos.x, pos.y, ball_r, playercolor, ball_segs )

end

local function maxy() 
    return res.y - playerh
end

local function randomUnitVector()
    local angle = random() * pi * 2
    local vx = cos(angle)
    local vy = sin(angle)
    return v2f{0.7 + random()*0.2, 0.2}
end
local ball_velocity = randomUnitVector() * (1200 + math.random() * 500)

local function reset_positions()
    res = res or imgui.get_display_size()
    playery = 0.5 * res.y
    enemyy = 0.5 * res.y
    ballpos =  res * 0.5
    ball_velocity = randomUnitVector()  * (1200 + math.random() * 500)
end



local keys = {
    --512
    "Tab", "LeftArrow", "RightArrow", "UpArrow", "DownArrow", "PageUp", "PageDown", "Home", "End",
    "Insert", "Delete", "Backspace", "Space", "Enter", "Escape", "LeftCtrl", "LeftShift", "LeftAlt", "WindowsKey",
    "RightCtrl", "RightShift", "RightAlt", "RightSuper", "Menu", "0", "1", "2", "3", "4", "5", "6", "7", "8", "9", "A",
    "B", "C", "D", "E", "F", "G", "H", "I", "J", "K", "L", "M", "N", "O", "P", "Q", "R", "S", "T", "U", "V", "W", "X",
    "Y", "Z", "F1", "F2", "F3", "F4", "F5", "F6", "F7", "F8", "F9", "F10", "F11", "F12", "Apostrophe", "Comma", "Minus",
    "Period", "Slash", "Semicolon", "Equal", "LeftBracket", "Backslash", "RightBracket", "GraveAccent", "CapsLock",
    "ScrollLock", "NumLock", "PrintScreen", "Pause", "Keypad0", "Keypad1", "Keypad2", "Keypad3", "Keypad4", "Keypad5",
    "Keypad6", "Keypad7", "Keypad8", "Keypad9", "KeypadDecimal", "KeypadDivide", "KeypadMultiply", "KeypadSubtract",
    "KeypadAdd", "KeypadEnter", "KeypadEqual", "GamepadStart",
    "GamepadBack", "GamepadFaceLeft", "GamepadFaceRight", "GamepadFaceUp", "GamepadFaceDown", "GamepadDpadLeft",
    "GamepadDpadRight",
    "GamepadDpadUp", "GamepadDpadDown", "GamepadL1", "GamepadR1", "GamepadL2", "GamepadR2", "GamepadL3", "GamepadR3",
    "GamepadLStickLeft",
    "GamepadLStickRight", "GamepadLStickUp", "GamepadLStickDown", "GamepadRStickLeft", "GamepadRStickRight",
    "GamepadRStickUp", "GamepadRStickDown",
    --641
    "MouseLeft", "MouseRight", "MouseMiddle", "MouseX1", "MouseX2", "MouseWheelX", "MouseWheelY"

}
local current_pressed_keys = {}
local current_released_keys = {}
local function update_keys()
    -- the real imgui enum values
    for i = 512, 647 do
        local key_name = keys[i - 511]
        if i ~= 530 then -- windows key
            local prev = current_pressed_keys[key_name] or false
            if imgui.is_key_down(i) then
                current_pressed_keys[key_name] = true
            else
                current_pressed_keys[key_name] = false
                if prev then
                    current_released_keys[key_name] = true
                else
                    current_released_keys[key_name] = false
                end
            end
        end
    end
end


local function player_movement(dt)
    local newy = playery
    if current_pressed_keys["W"] or  current_pressed_keys["GamepadLStickUp"]  then
        newy = math.max(0, playery - (dt * move_speed))

    end
    if current_pressed_keys["S"] or current_pressed_keys["GamepadLStickDown"]  then
        newy = math.min(playery + (dt * move_speed), maxy())
    end
    return newy
end


local function clamp(val, min, max)
    if val < min then return min end
    if val > max then return max end
    return val
end

local function player_rect()
    return {x = 0, y = playery, w = playerw, h = playerh}
end

local function enemy_rect()
    return {x = (res.x) - playerw, y = enemyy, w = playerw, h = playerh}
end

local function ceiling_rect()
    return {x = 0, y=0, w = res.x, h = 2}
end

local function floor_rect()
    return {x = 0, y=(res.y)-10, w = res.x, h = 2}
end

local function radial_distance_vec(vec, center)
    return (vec - center) * (1 / (vec - center):length())
end


local function collision(pos, rect, vel)
    local close = Vector2f.new(clamp(pos.x, rect.x, rect.x + rect.w), clamp(pos.y, rect.y, rect.y + rect.h))
    local distance = radial_distance_vec(pos, close)

    if distance < (ball_r ) then
        local normal = (pos - close) * (1 / distance)
        -- hitting a wall reduces speed, hitting a paddle increases
        -- there's no special logic here, I just hardcoded based on the sizes I set. bad practice tbh
        local bounce = (rect.h <= 10) and 0.9 or 1.1
        -- prevent clipping through objects
        local overlap = ball_r - distance
        local new_pos = Vector2f.new(pos.x + normal.x * overlap, pos.y + normal.y * overlap)

        return (pos + (normal * overlap)), (vel:reflect(normal) * bounce)
    end
    return pos, vel
end
local function update_pos(pos, vel, dt)
    -- Calculate movement
    local newpos = pos + (vel * dt)
    if not is_resetting then
        -- Score Left (Enemy wins point)
        if newpos.x <= 0 then
            Score[2] = Score[2] + 1
            is_resetting = true
            return pos, vel -- Stop movement
        -- Score Right (Player wins point)
        elseif newpos.x >= res.x then
            Score[1] = Score[1] + 1
            is_resetting = true
            return pos, vel
        end

        -- Ceiling/Floor Collisions
        if (newpos.y - ball_r) <= 0 then
            newpos, vel = collision(newpos, ceiling_rect(), vel)
        elseif (newpos.y + ball_r) >= res.y then
            newpos, vel = collision(newpos, floor_rect(), vel)
        end
    end

    return newpos, newvel
end

local function ball_movement(pos, vel, dt)
    local next_pos = pos + (vel * dt)
    local next_vel = vel

    -- 2. Wall Collisions (Top / Bottom)
    if next_pos.y - ball_r <= 0 or next_pos.y + ball_r >= res.y then
        next_vel = Vector2f.new(next_vel.x, -next_vel.y)
        -- Keep ball within bounds
        next_pos.y = clamp(next_pos.y, ball_r, res.y - ball_r)
    end


    -- left side collisions (player)
    if next_vel.x < 0 and next_pos.x - ball_r <= playerw then
        if next_pos.y >= playery and next_pos.y <= playery + playerh then
            next_vel = Vector2f.new(next_vel.x * -1.10, next_vel.y) -- Speed up 10%
            next_pos.x = playerw + ball_r
        end
    end

    -- right side collision (enemy)
    if next_vel.x > 0 and next_pos.x + ball_r >= (res.x - playerw) then
        if next_pos.y >= enemyy and next_pos.y <= enemyy + playerh then
            next_vel = Vector2f.new(-next_vel.x * 1.075, next_vel.y) -- Speed up 7.5%
            next_pos.x = res.x - playerw - ball_r
            ai_precision = 0.7 + (random() * 0.25)
        end
    end

    -- scoring + reset
    if next_pos.x <= 0 then
        Score[2] = Score[2] + 1
       reset_positions()
        is_resetting = true
    elseif next_pos.x >= res.x then
        Score[1] = Score[1] + 1
        reset_positions()
        is_resetting = true
    end

    return next_pos, next_vel
end

local function draw_player(y_pos)
    draw.filled_rect(0, math.min(y_pos, maxy()), playerw, playerh, playercolor)
end
local function draw_enemy(y_pos)
    draw.filled_rect(res.x - playerw, math.min(y_pos, maxy()), playerw, playerh, playercolor)
end



local function sign(x)
  return x < 0 and -1 or 1
end

pc = api:get_player_controller(0)
local kismet = uevr.api:find_uobject("Class /Script/Engine.KismetStringLibrary"):get_class_default_object()
local EControllerAnalogStick =
{
    CAS_LeftStick                            = 0,
    CAS_RightStick                           = 1,
    CAS_MAX                                  = 2,
}
local function get_left_stick()
    pc = pc or api:get_player_controller(0)
    local stickx, sticky = {}, {}
    pc:GetInputAnalogStickState(EControllerAnalogStick["CAS_LeftStick"], stickx, sticky)
    return {stickx.result, sticky.result}
end



local function enemy_ai(current_y, ball_pos, ball_vel, dt)
    -- add a bit of variety

    -- ball is moving towards enemy so actually run the logic
    if ball_vel.x > 0 then
        -- Calculate time to reach enemy paddle
        local dist_x = (res.x - playerw) - ball_pos.x
        local time_to_hit = dist_x / ball_vel.x

        -- this can only predict simple collision
        local predicted_y = ball_pos.y + (ball_vel.y * time_to_hit)

       -- add some error
        enemy_target_y = predicted_y + (random(-20, 20) * (1 - ai_precision))
    else
        -- ball is moving towards player, just aim for the center
        enemy_target_y = 0.5 * res.y
    end

    -- clamp within screen
    enemy_target_y = clamp(enemy_target_y, 0, maxy())


    local diff = enemy_target_y - (current_y + playerh / 2)
    if math.abs(diff) > 5 then
        local move_dir = diff > 0 and 1 or -1
        current_y = current_y + (move_dir * enemy_speed * dt)
    end

    return clamp(current_y, 0, maxy())
end

-- Converts HSV (0.0-1.0 range) to RGB (0.0-1.0 range)
function hsv_to_rgb(h, s, v)
    local i = floor(h * 6)
    local f = h * 6 - i
    local p = v * (1 - s)
    local q = v * (1 - f)
    local t = v * (1 - (1 - f) * s)

    local r, g, b
    local mod_i = i % 6

    local res =
    {
        {v, t, p},
        {q, v, p},
        {p, v, t},
        {p, q, v},
        {t, p, v},
    }

    if mod_i <= 4 then
        r, g, b = table.unpack(res[mod_i + 1])
    else r, g, b = v, p, q
    end

    return r, g, b
end

-- get a semi-random, bright, and saturated color (0.0-1.0 RGBA)
-- easy to use for debug shapes, e.y. with 800 or so colliders suddenly drawn
-- use VecToU32 with draw api
local function get_semi_random_bright_color(index, total_count)
    if index == nil and total_count == nil then
        index = random(255)
        total_count = 255
    end

    -- convert to HSV since this is easiest to think of in terms of hue variation by index
    -- (math.random() * 0.1) adds a jitter to reduce banding
    local h = (index / total_count + random() * 0.1) % 1.0

    -- hsv also makes it easier to guarantee high saturation without bias towards a hue
    local s = 0.8 + random() * 0.2

    -- same deal with brightness
    local v = 0.9 + random() * 0.1

    -- Convert HSV back to RGB
    local r, g, b = hsv_to_rgb(h, s, v)

    return v4f{r, g, b, 1.0}
end


local ui_text_colors = {VecToU32(get_semi_random_bright_color()), VecToU32(get_semi_random_bright_color())}
local was_open = false
local open = true
local frames = 0
local last_frame
local was_reset = false
local playing = false
local game_over = false
uevr.sdk.callbacks.on_frame(function()
    update_keys()
    local s, r = pcall(function()
    local dt = last_frame and (os.clock() - last_frame) or 0.001667
    if dt > 0.02 then dt = 0.01667 end

    res = res or imgui.get_display_size()
    imgui.set_next_window_size(res)
    imgui.set_next_window_pos(Vector2f.new(0, 0))

    imgui.push_style_color(2, Vector4f.new(0.0, 0.0, 0.0, 0.7))


    pc = pc or api:get_player_controller(0)
    world =  world or pc:get_outer():get_outer()


   local function draw_paddle(x, y, color)
    draw.filled_rect(x, y, playerw, playerh, color)
    end

    imgui.begin_window("Pong", true, 798751)
        if not is_resetting then
            playery = player_movement(dt)
            ballpos, ball_velocity = ball_movement(ballpos, ball_velocity, dt)
            enemyy = enemy_ai(enemyy, ballpos, ball_velocity, dt)
        else
            if current_released_keys["Space"] then
                is_resetting = false
            end
        end

        draw.filled_rect(0, 0, res.x, res.y, 0xAA000000) -- background

        draw_paddle(0, playery, playercolor)
        draw_paddle(res.x - playerw, enemyy, playercolor)
        draw_ball(ballpos)


        draw.filled_rect((res.x * 0.5) - 2, 0, (res.x * 0.5) + 2, res.y, 0x88FFFFFF)

        draw.text("Player: " .. Score[1], res.x * 0.25, 50, 0xFFFFFFFF)
        draw.text("Enemy: " .. Score[2], res.x * 0.75, 50, 0xFFFFFFFF)

    imgui.end_window()

    last_frame = os.clock()
    frames = frames + 1 
    end) if not s then print(r) end
end)

uevr.sdk.callbacks.on_pre_engine_tick(function(engine, delta)
    if not is_resetting then
        left_stick = get_left_stick()
    end
end)

return pong
```
