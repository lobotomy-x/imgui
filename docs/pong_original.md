# Pong — Original Lua Script (UEVR)

This is the original Pong game written in Lua for the
[UEVR](https://github.com/praydog/UEVR) (Unreal Engine VR) framework,
the starting point for the C++ port found in `imgui_demo.cpp` →
`ShowExampleAppPong()`.

Dead code from the first draft has been trimmed (the duplicate
`local pong`, the never-called `update_pos` / `collision` / `*_rect`
helpers, the unused HSV palette generator, the gamepad-stick read whose
value was never consumed, the `imgui.push_style_color` with no matching
pop, etc.). Behavior is unchanged — the same straight-line AI prediction
that the C++ port replaced with a reflection-aware projection is still
right here in `enemy_ai`.

---

```lua
local pong = {}

local v2f = function(t)
    return Vector2f.new(table.unpack(t))
end

local random = math.random

local res = imgui.get_display_size() or v2f{1920, 1080}
local playercolor = 0xFFDBDDDE
local playerh = 120
local playerw = 60

local ball_r = 24
local ball_segs = 18

local ballpos = res * 0.5
local enemyy = 0.5 * res.y
local playery = 0.5 * res.y
local Score = {0, 0}
local enemy_target_y = 0.5 * res.y
local enemy_speed = 1000     -- slightly slower than player so it's beatable
local ai_precision = 0.9     -- 1.0 = perfect, lower = more "human"
local move_speed = 1200
local is_resetting = true

local function maxy()
    return res.y - playerh
end

local function clamp(val, min, max)
    if val < min then return min end
    if val > max then return max end
    return val
end

-- Not actually a unit vector — a serve from the left at a shallow angle.
-- Name is preserved from the original for posterity.
local function randomUnitVector()
    return v2f{0.7 + random() * 0.2, 0.2}
end

local ball_velocity = randomUnitVector() * (1200 + random() * 500)

local function reset_positions()
    res = res or imgui.get_display_size()
    playery = 0.5 * res.y
    enemyy = 0.5 * res.y
    ballpos = res * 0.5
    ball_velocity = randomUnitVector() * (1200 + random() * 500)
end

-- ImGui integer key codes 512..647 mapped to friendly names. The polling
-- loop below writes current_pressed_keys / current_released_keys so game
-- logic can ask by name. We only consume W, S, Space, and the LStick Y axis,
-- but the full table keeps the loop trivial.
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
    for i = 512, 647 do
        local key_name = keys[i - 511]
        if i ~= 530 then -- skip WindowsKey
            local prev = current_pressed_keys[key_name] or false
            if imgui.is_key_down(i) then
                current_pressed_keys[key_name] = true
                current_released_keys[key_name] = false
            else
                current_pressed_keys[key_name] = false
                current_released_keys[key_name] = prev
            end
        end
    end
end

local function player_movement(dt)
    local newy = playery
    if current_pressed_keys["W"] or current_pressed_keys["GamepadLStickUp"] then
        newy = math.max(0, playery - dt * move_speed)
    end
    if current_pressed_keys["S"] or current_pressed_keys["GamepadLStickDown"] then
        newy = math.min(playery + dt * move_speed, maxy())
    end
    return newy
end

local function ball_movement(pos, vel, dt)
    local next_pos = pos + vel * dt
    local next_vel = vel

    -- top / bottom wall reflection
    if next_pos.y - ball_r <= 0 or next_pos.y + ball_r >= res.y then
        next_vel = Vector2f.new(next_vel.x, -next_vel.y)
        next_pos.y = clamp(next_pos.y, ball_r, res.y - ball_r)
    end

    -- player paddle (left)
    if next_vel.x < 0 and next_pos.x - ball_r <= playerw then
        if next_pos.y >= playery and next_pos.y <= playery + playerh then
            next_vel = Vector2f.new(next_vel.x * -1.10, next_vel.y) -- +10% speed
            next_pos.x = playerw + ball_r
        end
    end

    -- enemy paddle (right)
    if next_vel.x > 0 and next_pos.x + ball_r >= res.x - playerw then
        if next_pos.y >= enemyy and next_pos.y <= enemyy + playerh then
            next_vel = Vector2f.new(-next_vel.x * 1.075, next_vel.y) -- +7.5% speed
            next_pos.x = res.x - playerw - ball_r
            ai_precision = 0.7 + random() * 0.25
        end
    end

    -- scoring
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

local function enemy_ai(current_y, ball_pos, ball_vel, dt)
    if ball_vel.x > 0 then
        -- Straight-line prediction — ignores wall bounces. (The C++ port
        -- replaced this with a reflection-aware projection so angled shots
        -- aren't free points anymore.)
        local time_to_hit = ((res.x - playerw) - ball_pos.x) / ball_vel.x
        local predicted_y = ball_pos.y + ball_vel.y * time_to_hit
        enemy_target_y = predicted_y + random(-20, 20) * (1 - ai_precision)
    else
        enemy_target_y = 0.5 * res.y
    end
    enemy_target_y = clamp(enemy_target_y, 0, maxy())

    local diff = enemy_target_y - (current_y + playerh / 2)
    if math.abs(diff) > 5 then
        current_y = current_y + (diff > 0 and 1 or -1) * enemy_speed * dt
    end
    return clamp(current_y, 0, maxy())
end

local function draw_player(y_pos)
    draw.filled_rect(0, math.min(y_pos, maxy()), playerw, playerh, playercolor)
end

local function draw_enemy(y_pos)
    draw.filled_rect(res.x - playerw, math.min(y_pos, maxy()), playerw, playerh, playercolor)
end

local function draw_ball(pos)
    draw.filled_circle(pos.x, pos.y, ball_r, playercolor, ball_segs)
end

local last_frame
uevr.sdk.callbacks.on_frame(function()
    update_keys()
    local ok, err = pcall(function()
        local dt = last_frame and (os.clock() - last_frame) or 0.001667
        if dt > 0.02 then dt = 0.01667 end

        res = res or imgui.get_display_size()
        imgui.set_next_window_size(res)
        imgui.set_next_window_pos(Vector2f.new(0, 0))

        imgui.begin_window("Pong", true, 798751)
            if not is_resetting then
                playery = player_movement(dt)
                ballpos, ball_velocity = ball_movement(ballpos, ball_velocity, dt)
                enemyy  = enemy_ai(enemyy, ballpos, ball_velocity, dt)
            elseif current_released_keys["Space"] then
                is_resetting = false
            end

            draw.filled_rect(0, 0, res.x, res.y, 0xAA000000) -- background
            draw_player(playery)
            draw_enemy(enemyy)
            draw_ball(ballpos)

            -- center divider
            draw.filled_rect((res.x * 0.5) - 2, 0, (res.x * 0.5) + 2, res.y, 0x88FFFFFF)

            draw.text("Player: " .. Score[1], res.x * 0.25, 50, 0xFFFFFFFF)
            draw.text("Enemy: "  .. Score[2], res.x * 0.75, 50, 0xFFFFFFFF)
        imgui.end_window()

        last_frame = os.clock()
    end)
    if not ok then print(err) end
end)

return pong
```
