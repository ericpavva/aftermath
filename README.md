local VERSION = "1.999"

if not draw or not utility or not camera or not input or not thread then
    if notify and notify.Error then notify.Error("Aftermath", "Incompatible Vector build") end
    return
end

local sqrt, floor, min, max = math.sqrt, math.floor, math.min, math.max
local abs, sin, cos = math.abs, math.sin, math.cos
local atan2 = (math.atan2 or math.atan)
local format = string.format
local STUDS_PER_METER = 2.6

local BOX_TYPE = {STANDARD = 0, CORNER = 1, THREE_D = 2}
local SNAP_MODE = {TOP = 0, CENTER = 1, BOTTOM = 2}
local LINE_STYLE = {SOLID = 0, DASHED = 1, FADE = 2}

local DS_OFFSET_FB = 7
local DS_OFFSET_LR = -1
local DS_CIRCLE_RADIUS = 4
local DS_CIRCLE_SEGMENTS = 64
local DS_DEFAULT_TOGGLE = 0x58
local DS_DEFAULT_MODE = 0x56

local GAME_GRAVITY_STUDS = 85
pcall(function()
    if game.Workspace and game.Workspace.Gravity then
        GAME_GRAVITY_STUDS = game.Workspace.Gravity
    end
end)
local GAME_GRAVITY_MPS = GAME_GRAVITY_STUDS / STUDS_PER_METER

-- ═══════════════════════════════════════════════════════════════════════
-- UI ENGINE
-- ═══════════════════════════════════════════════════════════════════════

local UI = {}
UI.__index = UI

UI.COLORS_DEFAULT = {
    bg          = {0.05, 0.06, 0.10, 0.96},
    titlebar    = {0.08, 0.10, 0.16, 1.00},
    border      = {0.16, 0.22, 0.34, 1.00},
    border_soft = {0.12, 0.16, 0.24, 1.00},
    accent      = {0.25, 0.60, 1.00, 1.00},
    accent_dim  = {0.25, 0.60, 1.00, 0.25},
    text        = {0.92, 0.95, 1.00, 1.00},
    text_dim    = {0.55, 0.60, 0.72, 1.00},
    text_muted  = {0.35, 0.40, 0.50, 1.00},
    panel       = {0.08, 0.10, 0.15, 1.00},
    panel_alt   = {0.11, 0.14, 0.21, 1.00},
    hover       = {0.16, 0.22, 0.34, 1.00},
    green       = {0.30, 0.95, 0.45, 1.00},
    red         = {1.00, 0.35, 0.35, 1.00},
}
UI.COLORS = {}
for k, v in pairs(UI.COLORS_DEFAULT) do UI.COLORS[k] = {v[1], v[2], v[3], v[4]} end

UI.state = {
    open = true,
    x = 80, y = 60, w = 860, h = 640,
    dragging = false, drag_ox = 0, drag_oy = 0,
    active_tab = 1,
    tabs = {},
    values = {}, colors = {}, keys = {},
    callbacks = {}, visible = {},
    open_combo = nil, open_combo_rect = nil,
    picker = nil, drag_target = nil,
    combo_just_opened = false,
    picker_just_opened = false,
    listening_key = nil, listen_wait_lmb_up = false,
    prev_lmb = false,
    toggle_key = 0x2D,
    scroll = 0,
    ui_settings = {
        accent      = {0.25, 0.60, 1.00, 1.00},
        opacity     = 96,
        window_w    = 860,
        window_h    = 640,
    },
}

function UI:apply_ui_settings()
    local us = self.state.ui_settings
    local base = self.COLORS_DEFAULT
    local acc = us.accent
    local op = (us.opacity or 96) / 100
    for k, v in pairs(base) do
        if type(v) == "table" then
            self.COLORS[k] = {v[1], v[2], v[3], (v[4] or 1) * op}
        end
    end
    self.COLORS.accent     = {acc[1], acc[2], acc[3], 1.00}
    self.COLORS.accent_dim = {acc[1], acc[2], acc[3], 0.25}
    self.state.w = us.window_w or 860
    self.state.h = us.window_h or 640
end

UI:apply_ui_settings()

local VK_NAMES = {
    [0x01] = "LMB", [0x02] = "RMB", [0x04] = "MMB",
    [0x08] = "Backspace", [0x09] = "Tab", [0x0D] = "Enter",
    [0x10] = "Shift", [0x11] = "Ctrl", [0x12] = "Alt",
    [0x13] = "Pause", [0x14] = "CapsLock", [0x1B] = "Escape",
    [0x20] = "Space", [0x21] = "PageUp", [0x22] = "PageDown",
    [0x23] = "End", [0x24] = "Home",
    [0x25] = "Left", [0x26] = "Up", [0x27] = "Right", [0x28] = "Down",
    [0x2C] = "PrintScr", [0x2D] = "Insert", [0x2E] = "Delete",
    [0x30] = "0", [0x31] = "1", [0x32] = "2", [0x33] = "3", [0x34] = "4",
    [0x35] = "5", [0x36] = "6", [0x37] = "7", [0x38] = "8", [0x39] = "9",
    [0x41] = "A", [0x42] = "B", [0x43] = "C", [0x44] = "D", [0x45] = "E",
    [0x46] = "F", [0x47] = "G", [0x48] = "H", [0x49] = "I", [0x4A] = "J",
    [0x4B] = "K", [0x4C] = "L", [0x4D] = "M", [0x4E] = "N", [0x4F] = "O",
    [0x50] = "P", [0x51] = "Q", [0x52] = "R", [0x53] = "S", [0x54] = "T",
    [0x55] = "U", [0x56] = "V", [0x57] = "W", [0x58] = "X", [0x59] = "Y",
    [0x5A] = "Z",
    [0x60] = "Num0", [0x61] = "Num1", [0x62] = "Num2", [0x63] = "Num3",
    [0x64] = "Num4", [0x65] = "Num5", [0x66] = "Num6", [0x67] = "Num7",
    [0x68] = "Num8", [0x69] = "Num9",
    [0x6A] = "Num*", [0x6B] = "Num+", [0x6D] = "Num-", [0x6E] = "Num.",
    [0x6F] = "Num/",
    [0x70] = "F1", [0x71] = "F2", [0x72] = "F3", [0x73] = "F4",
    [0x74] = "F5", [0x75] = "F6", [0x76] = "F7", [0x77] = "F8",
    [0x78] = "F9", [0x79] = "F10", [0x7A] = "F11", [0x7B] = "F12",
    [0x90] = "NumLock", [0x91] = "ScrollLock",
    [0xA0] = "LShift", [0xA1] = "RShift", [0xA2] = "LCtrl", [0xA3] = "RCtrl",
    [0xA4] = "LAlt", [0xA5] = "RAlt",
    [0xBA] = ";", [0xBB] = "=", [0xBC] = ",", [0xBD] = "-",
    [0xBE] = ".", [0xBF] = "/", [0xC0] = "`",
    [0xDB] = "[", [0xDC] = "\\", [0xDD] = "]", [0xDE] = "'",
}

local function vk_name(vk)
    if not vk or vk == 0 then return "[none]" end
    return "[" .. (VK_NAMES[vk] or ("0x" .. format("%02X", vk))) .. "]"
end

local function in_rect(mx, my, x, y, w, h)
    return mx >= x and mx <= x + w and my >= y and my <= y + h
end

local function mouse_pos()
    if utility and utility.GetMousePos then
        local ok, mx, my = pcall(utility.GetMousePos)
        if ok and mx and my then return mx, my end
    end
    if input and input.GetMousePosition then
        local ok, mx, my = pcall(input.GetMousePosition)
        if ok and mx and my then return mx, my end
    end
    return 0, 0
end

local function key_down(vk)
    if input and input.IsKeyDown then
        local ok, d = pcall(input.IsKeyDown, vk)
        return ok and d == true
    end
    return false
end

local function text_w(s, size)
    if draw and draw.GetTextSize then
        local ok, w = pcall(draw.GetTextSize, tostring(s or ""), size or 13)
        if ok and w then return w end
    end
    return #tostring(s or "") * (size or 13) * 0.55
end

function UI:add_tab(name)
    self.state.tabs[#self.state.tabs + 1] = { name = name, groups = {} }
end

function UI:add_group(tab_name, group_name)
    for _, t in ipairs(self.state.tabs) do
        if t.name == tab_name then
            for _, g in ipairs(t.groups) do
                if g.name == group_name then return end
            end
            t.groups[#t.groups + 1] = { name = group_name, widgets = {} }
            return
        end
    end
    self:add_tab(tab_name)
    self:add_group(tab_name, group_name)
end

local function reg(self, tab, group, id, data)
    for _, t in ipairs(self.state.tabs) do
        if t.name == tab then
            for _, g in ipairs(t.groups) do
                if g.name == group then
                    data.id = id
                    g.widgets[#g.widgets + 1] = data
                    return
                end
            end
        end
    end
    self:add_group(tab, group)
    reg(self, tab, group, id, data)
end

function UI:add_checkbox(tab, group, id, label, default, opts)
    opts = opts or {}
    self.state.values[id] = default == true
    if opts.colorpicker then self.state.colors[id] = opts.colorpicker end
    reg(self, tab, group, id, { type="checkbox", label=label, default=default==true, color=opts.colorpicker, parent=opts.parent })
end

function UI:add_slider_int(tab, group, id, label, mn, mx, default, opts)
    opts = opts or {}
    self.state.values[id] = default or mn
    reg(self, tab, group, id, { type="slider_int", label=label, min=mn, max=mx, default=default, parent=opts.parent, fmt="%d" })
end

function UI:add_slider_float(tab, group, id, label, mn, mx, default, fmt, opts)
    opts = opts or {}
    self.state.values[id] = default or mn
    reg(self, tab, group, id, { type="slider_float", label=label, min=mn, max=mx, default=default, parent=opts.parent, fmt=fmt or "%.2f" })
end

function UI:add_combo(tab, group, id, label, items, default_idx, opts)
    opts = opts or {}
    self.state.values[id] = default_idx or 0
    reg(self, tab, group, id, { type="combo", label=label, items=items, default=default_idx or 0, parent=opts.parent })
end

function UI:add_multicombo(tab, group, id, label, items, defaults, opts)
    opts = opts or {}
    local def = {}
    for i = 1, #items do def[i] = defaults and defaults[i] == true end
    self.state.values[id] = def
    reg(self, tab, group, id, { type="multi", label=label, items=items, defaults=defaults, parent=opts.parent })
end

function UI:add_colorpicker(tab, group, id, label, default, opts)
    opts = opts or {}
    self.state.colors[id] = default or {1,1,1,1}
    reg(self, tab, group, id, { type="color", label=label, default=default, parent=opts.parent })
end

function UI:add_button(tab, group, id, label, callback)
    self.state.callbacks[id] = callback
    reg(self, tab, group, id, { type="button", label=label })
end

function UI:add_hotkey(tab, group, id, label, default_key, opts)
    opts = opts or {}
    self.state.keys[id] = default_key or 0
    reg(self, tab, group, id, { type="hotkey", label=label, default=default_key, parent=opts.parent })
end

function UI:add_label(tab, group, text)
    reg(self, tab, group, "__lbl_" .. tostring(math.random(999999)), { type="label", label=text })
end

function UI:get(id) return self.state.values[id] end
function UI:set(id, v)
    self.state.values[id] = v
    local cb = self.state.callbacks[id]
    if cb then pcall(cb, v) end
end
function UI:get_color(id) return self.state.colors[id] or {1,1,1,1} end
function UI:set_color(id, c) self.state.colors[id] = c end
function UI:get_key(id) return self.state.keys[id] or 0 end
function UI:set_key(id, k) self.state.keys[id] = k end
function UI:set_callback(id, cb) self.state.callbacks[id] = cb end
function UI:set_visible(id, v) self.state.visible[id] = v == true end

local function hsv_to_rgb(h, s, v)
    h = h * 6
    local i = floor(h)
    local f = h - i
    local p, q, t = v*(1-s), v*(1-f*s), v*(1-(1-f)*s)
    if i == 0 then return v, t, p end
    if i == 1 then return q, v, p end
    if i == 2 then return p, v, t end
    if i == 3 then return p, q, v end
    if i == 4 then return t, p, v end
    return v, p, q
end

local function rgb_to_hsv(r, g, b)
    local mx = max(r, g, b)
    local mn = min(r, g, b)
    local d = mx - mn
    local h = 0
    if d > 1e-6 then
        if mx == r then h = ((g - b) / d) % 6
        elseif mx == g then h = (b - r) / d + 2
        else h = (r - g) / d + 4 end
        h = h / 6
    end
    local s = (mx > 0) and (d / mx) or 0
    return h, s, mx
end

local picker_state = {
    hue = nil, sat = nil, val = nil,
    dragging_sv = false, dragging_hue = false,
}

local function draw_color_picker(self, id, x, y, w, h)
    local c = self:get_color(id)
    draw.RectFilled(x + 3, y + 3, w - 6, h - 6, {0,0,0,0.5}, 6)
    local sq = min(w - 60, h - 60)
    local sx, sy = x + 12, y + 30
    local hue, sat, val = rgb_to_hsv(c[1], c[2], c[3])
    if picker_state.hue then
        hue, sat, val = picker_state.hue, picker_state.sat, picker_state.val
    end
    local steps = 12
    local cell = sq / steps
    for iy = 0, steps - 1 do
        for ix = 0, steps - 1 do
            local s = ix / (steps - 1)
            local v = 1 - iy / (steps - 1)
            local r, g, b = hsv_to_rgb(hue, s, v)
            draw.RectFilled(sx + ix*cell, sy + iy*cell, cell + 0.5, cell + 0.5, {r,g,b,1}, 0)
        end
    end
    draw.Rect(sx, sy, sq, sq, self.COLORS.border, 0, 1)
    local hx, hy, hw, hh = sx + sq + 8, sy, 14, sq
    for i = 0, 17 do
        local t = i / 17
        local r, g, b = hsv_to_rgb(t, 1, 1)
        draw.RectFilled(hx, hy + i*(hh/18), hw, hh/18 + 0.5, {r,g,b,1}, 0)
    end
    draw.Rect(hx, hy, hw, hh, self.COLORS.border, 0, 1)
    local sv_cursor_x = sx + sat * sq
    local sv_cursor_y = sy + (1 - val) * sq
    draw.Circle(sv_cursor_x, sv_cursor_y, 6, {1,1,1,1}, 16, 2)
    draw.Circle(sv_cursor_x, sv_cursor_y, 6, {0,0,0,1}, 16, 1)
    local hue_y = hy + hue * hh
    draw.RectFilled(hx - 2, hue_y - 2, hw + 4, 4, {1,1,1,1}, 2)
    draw.Rect(hx - 2, hue_y - 2, hw + 4, 4, {0,0,0,1}, 2, 1)
    local px, py = x + 12, sy + sq + 8
    draw.RectFilled(px, py, sq, 20, c, 4)
    draw.Rect(px, py, sq, 20, self.COLORS.border, 4, 1)
    local bx, by, bw, bh = x + w - 78, py, 66, 20
    draw.RectFilled(bx, by, bw, bh, self.COLORS.accent, 4)
    draw.Text(bx + 22, by + 4, "OK", {1,1,1,1}, 12)

    local mx, my = mouse_pos()
    local lmb = key_down(0x01)

    if not lmb then
        picker_state.dragging_sv = false
        picker_state.dragging_hue = false
    end

    if self.state.picker_just_opened then return end

    if lmb then
        if picker_state.dragging_sv or in_rect(mx, my, sx, sy, sq, sq) then
            picker_state.dragging_sv = true
            sat = max(0, min(1, (mx - sx) / sq))
            val = max(0, min(1, 1 - (my - sy) / sq))
            local r, g, b = hsv_to_rgb(hue, sat, val)
            c[1], c[2], c[3] = r, g, b
            picker_state.hue, picker_state.sat, picker_state.val = hue, sat, val
            self:set_color(id, c)
            if self.state.callbacks[id] then pcall(self.state.callbacks[id], c) end
        elseif picker_state.dragging_hue or in_rect(mx, my, hx, hy, hw, hh) then
            picker_state.dragging_hue = true
            hue = max(0, min(1, (my - hy) / hh))
            local r, g, b = hsv_to_rgb(hue, sat, val)
            c[1], c[2], c[3] = r, g, b
            picker_state.hue, picker_state.sat, picker_state.val = hue, sat, val
            self:set_color(id, c)
            if self.state.callbacks[id] then pcall(self.state.callbacks[id], c) end
        elseif in_rect(mx, my, bx, by, bw, bh) then
            self.state.picker = nil
            picker_state.hue = nil
        end
    end
end

local function draw_widget(self, w, x, y, wd, h, lmb_down, lmb_click, block_input)
    if w.type == "label" then
        draw.Text(x + 4, y + (h - 12)/2, w.label, self.COLORS.text_muted, 11)
        return
    end

    local mx, my = mouse_pos()
    local hovered = in_rect(mx, my, x, y, wd, h) and not block_input

    if w.type == "checkbox" then
        local on = self.state.values[w.id] == true
        if hovered then draw.RectFilled(x, y, wd, h, self.COLORS.hover, 4) end
        local sw, sh = 30, 16
        local sx, sy = x + 4, y + (h - sh) / 2
        local track = on and self.COLORS.accent or self.COLORS.border_soft
        draw.RectFilled(sx, sy, sw, sh, track, sh/2)
        local knob = sh - 4
        local kx = on and (sx + sw - knob - 2) or (sx + 2)
        draw.CircleFilled(kx + knob/2, sy + sh/2, knob/2, self.COLORS.text, 16)
        draw.Text(sx + sw + 8, y + (h - 13)/2, w.label,
            on and self.COLORS.text or self.COLORS.text_dim, 13)
        if self.state.colors[w.id] ~= nil then
            local c = self:get_color(w.id)
            local cx = x + wd - 20
            local cy = y + (h - 14) / 2
            draw.RectFilled(cx, cy, 14, 14, c, 4)
            draw.Rect(cx, cy, 14, 14, self.COLORS.border, 4, 1)
            if lmb_click and hovered and in_rect(mx, my, cx - 3, cy - 3, 20, 20) then
                self.state.picker = { id = w.id, x = cx - 100, y = y + h + 4, w = 200, h = 200 }
                self.state.picker_just_opened = true
                picker_state.hue = nil
            end
        end
        if lmb_click and hovered and not (self.state.colors[w.id] and in_rect(mx, my, x + wd - 20, y, 20, h)) then
            self.state.values[w.id] = not on
            if self.state.callbacks[w.id] then pcall(self.state.callbacks[w.id], self.state.values[w.id]) end
        end

    elseif w.type == "slider_int" or w.type == "slider_float" then
        local v = tonumber(self.state.values[w.id]) or w.default or w.min
        local slider_y = y + h - 9
        local slider_h = 5
        local slider_w = wd - 8
        local track_x = x + 4
        local fmt = w.fmt or "%d"
        local vtxt = format(fmt, v)
        local vw = text_w(vtxt, 12)
        draw.Text(x + 4, y + 2, w.label, self.COLORS.text, 12)
        draw.Text(x + wd - vw - 4, y + 2, vtxt, self.COLORS.accent, 12)
        draw.RectFilled(track_x, slider_y, slider_w, slider_h, self.COLORS.border_soft, slider_h/2)
        local t = (w.max > w.min) and ((v - w.min) / (w.max - w.min)) or 0
        draw.RectFilled(track_x, slider_y, slider_w * t, slider_h, self.COLORS.accent, slider_h/2)
        draw.CircleFilled(track_x + slider_w * t, slider_y + slider_h/2, 6, self.COLORS.text, 16)
        local hot = in_rect(mx, my, track_x, slider_y - 6, slider_w, slider_h + 12) and not block_input
        if lmb_down then
            if not self.state.drag_target and hot then self.state.drag_target = w.id end
            if self.state.drag_target == w.id then
                local nt = max(0, min(1, (mx - track_x) / slider_w))
                local nv = w.min + (w.max - w.min) * nt
                if w.type == "slider_int" then nv = floor(nv + 0.5) end
                self.state.values[w.id] = nv
                if self.state.callbacks[w.id] then pcall(self.state.callbacks[w.id], nv) end
            end
        else
            if self.state.drag_target == w.id then self.state.drag_target = nil end
        end

    elseif w.type == "combo" then
        local idx = tonumber(self.state.values[w.id]) or 0
        if hovered then draw.RectFilled(x, y, wd, h, self.COLORS.hover, 4) end
        draw.Text(x + 6, y + (h - 13)/2, w.label, self.COLORS.text_dim, 12)
        local cur = w.items[idx + 1] or "-"
        local cw = text_w(cur, 12)
        draw.Text(x + wd - cw - 16, y + (h - 13)/2, cur, self.COLORS.text, 12)
        draw.Text(x + wd - 10, y + (h - 13)/2, "v", self.COLORS.text_dim, 10)
        if lmb_click and hovered then
            if self.state.open_combo == w.id then
                self.state.open_combo = nil
                self.state.open_combo_rect = nil
            else
                self.state.open_combo = w.id
                self.state.combo_just_opened = true
                self.state.open_combo_rect = { x = x, y = y + h, w = wd, widget = w }
            end
        end

    elseif w.type == "multi" then
        local vals = self.state.values[w.id] or {}
        if hovered then draw.RectFilled(x, y, wd, h, self.COLORS.hover, 4) end
        draw.Text(x + 6, y + (h - 13)/2, w.label, self.COLORS.text_dim, 12)
        local n = 0
        for i = 1, #w.items do if vals[i] then n = n + 1 end end
        local txt = (n == 0) and "None" or (n .. " selected")
        local tw = text_w(txt, 12)
        draw.Text(x + wd - tw - 8, y + (h - 13)/2, txt, self.COLORS.accent, 12)
        if lmb_click and hovered then
            if self.state.open_combo == w.id then
                self.state.open_combo = nil
                self.state.open_combo_rect = nil
            else
                self.state.open_combo = w.id
                self.state.combo_just_opened = true
                self.state.open_combo_rect = { x = x, y = y + h, w = wd, widget = w }
            end
        end

    elseif w.type == "button" then
        local bg = hovered and self.COLORS.hover or self.COLORS.panel_alt
        draw.RectFilled(x, y + 2, wd, h - 4, bg, 4)
        draw.Rect(x, y + 2, wd, h - 4, self.COLORS.border_soft, 4, 1)
        local tw = text_w(w.label, 12)
        draw.Text(x + (wd - tw)/2, y + (h - 13)/2, w.label, self.COLORS.text, 12)
        if lmb_click and hovered then
            if self.state.callbacks[w.id] then pcall(self.state.callbacks[w.id]) end
        end

    elseif w.type == "hotkey" then
        local k = self.state.keys[w.id] or 0
        if hovered then draw.RectFilled(x, y, wd, h, self.COLORS.hover, 4) end
        draw.Text(x + 6, y + (h - 13)/2, w.label, self.COLORS.text, 12)
        local kname
        if self.state.listening_key == w.id then kname = "[press...]"
        else kname = vk_name(k) end
        local kw = text_w(kname, 12)
        draw.RectFilled(x + wd - kw - 14, y + 4, kw + 10, h - 8, self.COLORS.panel_alt, 4)
        draw.Rect(x + wd - kw - 14, y + 4, kw + 10, h - 8, self.COLORS.border_soft, 4, 1)
        draw.Text(x + wd - kw - 9, y + (h - 13)/2, kname, self.COLORS.accent, 12)
        if lmb_click and hovered then
            self.state.listening_key = w.id
            self.state.listen_wait_lmb_up = true
        end

    elseif w.type == "color" then
        local c = self:get_color(w.id)
        if hovered then draw.RectFilled(x, y, wd, h, self.COLORS.hover, 4) end
        draw.Text(x + 6, y + (h - 13)/2, w.label, self.COLORS.text, 12)
        local cx = x + wd - 24
        draw.RectFilled(cx, y + 4, 16, h - 8, c, 3)
        draw.Rect(cx, y + 4, 16, h - 8, self.COLORS.border, 3, 1)
        if lmb_click and in_rect(mx, my, cx - 3, y, 22, h) then
            self.state.picker = { id = w.id, x = cx - 100, y = y + h + 4, w = 200, h = 200 }
            self.state.picker_just_opened = true
            picker_state.hue = nil
        end
    end
end

local function draw_open_combo_overlay(self)
    if not self.state.open_combo then return end
    local r = self.state.open_combo_rect
    if not r then return end
    local w = r.widget
    if not w then return end

    local mx, my = mouse_pos()
    local lmb_down = key_down(0x01)
    local lmb_click = lmb_down and not self.state.prev_lmb

    local x, y, wd = r.x, r.y, r.w
    local item_h = 20
    local list_h = #w.items * item_h

    if w.type == "combo" then
        local idx = tonumber(self.state.values[w.id]) or 0
        draw.RectFilled(x, y, wd, list_h, self.COLORS.panel_alt, 4)
        draw.Rect(x, y, wd, list_h, self.COLORS.border, 4, 1)
        for i, item in ipairs(w.items) do
            local iy = y + (i - 1) * item_h
            local ih = in_rect(mx, my, x, iy, wd, item_h)
            if ih then draw.RectFilled(x + 2, iy + 1, wd - 4, item_h - 2, self.COLORS.hover, 2) end
            local c = (i - 1 == idx) and self.COLORS.accent or self.COLORS.text
            draw.Text(x + 8, iy + 4, item, c, 12)
            if lmb_click and ih and not self.state.combo_just_opened then
                self.state.values[w.id] = i - 1
                self.state.open_combo = nil
                self.state.open_combo_rect = nil
                if self.state.callbacks[w.id] then pcall(self.state.callbacks[w.id], i - 1) end
            end
        end

    elseif w.type == "multi" then
        local vals = self.state.values[w.id] or {}
        draw.RectFilled(x, y, wd, list_h, self.COLORS.panel_alt, 4)
        draw.Rect(x, y, wd, list_h, self.COLORS.border, 4, 1)
        for i, item in ipairs(w.items) do
            local iy = y + (i - 1) * item_h
            local ih = in_rect(mx, my, x, iy, wd, item_h)
            if ih then draw.RectFilled(x + 2, iy + 1, wd - 4, item_h - 2, self.COLORS.hover, 2) end
            draw.RectFilled(x + 6, iy + 5, 10, 10, self.COLORS.border_soft, 2)
            if vals[i] then draw.RectFilled(x + 8, iy + 7, 6, 6, self.COLORS.accent, 1) end
            draw.Text(x + 22, iy + 4, item, self.COLORS.text, 12)
            if lmb_click and ih and not self.state.combo_just_opened then
                vals[i] = not vals[i]
                self.state.values[w.id] = vals
                if self.state.callbacks[w.id] then pcall(self.state.callbacks[w.id], vals) end
            end
        end
    end

    if lmb_click and not self.state.combo_just_opened then
        if not in_rect(mx, my, x, y, wd, list_h) then
            self.state.open_combo = nil
            self.state.open_combo_rect = nil
        end
    end
end

local function draw_window(self)
    local st = self.state
    if not st.open then return end

    local mx, my = mouse_pos()
    local lmb_down = key_down(0x01)
    local lmb_click = lmb_down and not st.prev_lmb

    draw.RectFilled(st.x + 4, st.y + 4, st.w, st.h, {0,0,0,0.35}, 6)
    draw.RectFilled(st.x, st.y, st.w, st.h, self.COLORS.bg, 6)
    draw.Rect(st.x, st.y, st.w, st.h, self.COLORS.border, 6, 1.5)

    local th = 34
    draw.RectFilled(st.x + 1, st.y + 1, st.w - 2, th, self.COLORS.titlebar, 5)
    draw.Line(st.x, st.y + th, st.x + st.w, st.y + th, self.COLORS.border_soft, 1)
    draw.Text(st.x + 14, st.y + 9, "AFTERMATH V2", self.COLORS.accent, 16)
    local vtxt = "v" .. VERSION
    local vw = text_w(vtxt, 11)
    draw.Text(st.x + st.w - vw - 14, st.y + 11, vtxt, self.COLORS.text_muted, 11)

    if lmb_down then
        if not st.dragging and in_rect(mx, my, st.x, st.y, st.w, th) then
            st.dragging = true
            st.drag_ox = mx - st.x
            st.drag_oy = my - st.y
        elseif st.dragging then
            st.x = mx - st.drag_ox
            st.y = my - st.drag_oy
        end
    else
        st.dragging = false
    end

    local tab_y = st.y + th
    local tab_h = 34
    draw.RectFilled(st.x + 1, tab_y, st.w - 2, tab_h, self.COLORS.panel, 0)
    local tx = st.x + 8
    for i, tab in ipairs(st.tabs) do
        local tw = text_w(tab.name, 13) + 32
        local active = (i == st.active_tab)
        local hovered = in_rect(mx, my, tx, tab_y + 4, tw, tab_h - 8)
        if active then
            draw.RectFilled(tx, tab_y + 4, tw, tab_h - 8, self.COLORS.accent_dim, 4)
            draw.RectFilled(tx, tab_y + tab_h - 3, tw, 2, self.COLORS.accent, 0)
        elseif hovered then
            draw.RectFilled(tx, tab_y + 4, tw, tab_h - 8, self.COLORS.hover, 4)
        end
        local c = active and self.COLORS.text or self.COLORS.text_dim
        draw.Text(tx + 16, tab_y + 12, tab.name, c, 13)
        if lmb_click and hovered and not st.combo_just_opened and not st.picker_just_opened then
            st.active_tab = i
            st.scroll = 0
            st.open_combo = nil
            st.open_combo_rect = nil
        end
        tx = tx + tw + 4
    end
    draw.Line(st.x, tab_y + tab_h, st.x + st.w, tab_y + tab_h, self.COLORS.border_soft, 1)

    local body_y = tab_y + tab_h + 4
    local body_bottom = st.y + st.h - 8
    local cur_tab = st.tabs[st.active_tab]
    if not cur_tab then
        st.prev_lmb = lmb_down
        return
    end

    -- se algum dropdown está aberto, bloqueia input dos widgets por baixo
    local block_input = (st.open_combo ~= nil) or (st.picker ~= nil)

    local pad = 8
    local col_w = floor((st.w - pad * 3) / 2)
    local cols = { st.x + pad, st.x + pad * 2 + col_w }

    local group_heights = {}
    local col_h = { 0, 0 }
    local col_assign = {}
    for i, g in ipairs(cur_tab.groups) do
        local hh = 26 + 10
        for _, w in ipairs(g.widgets) do
            if self.state.visible[w.id] ~= false then hh = hh + 26 + 4 end
        end
        group_heights[i] = hh
        local ci = (col_h[1] <= col_h[2]) and 1 or 2
        col_assign[i] = ci
        col_h[ci] = col_h[ci] + hh + 10
    end
    local total_h = max(col_h[1], col_h[2]) + pad * 2
    local view_h = body_bottom - body_y
    local max_scroll = max(0, total_h - view_h)
    st.scroll = st.scroll or 0
    st.scroll = max(0, min(max_scroll, st.scroll))

    local ys = { body_y + pad - st.scroll, body_y + pad - st.scroll }

    for i, g in ipairs(cur_tab.groups) do
        local ci = col_assign[i]
        local gx = cols[ci]
        local gy = ys[ci]

        local header_h = 26
        local row_h = 26
        local content_h = 0
        for _, w in ipairs(g.widgets) do
            if self.state.visible[w.id] ~= false then
                content_h = content_h + row_h + 4
            end
        end
        local group_h = header_h + content_h + 10

        if gy < body_bottom and gy + group_h > body_y then
            local clip_top = max(gy, body_y)
            local clip_bottom = min(gy + group_h, body_bottom)
            local clip_h = clip_bottom - clip_top
            if clip_h > 0 then
                draw.RectFilled(gx, clip_top, col_w, clip_h, self.COLORS.panel, 5)
                draw.Rect(gx, clip_top, col_w, clip_h, self.COLORS.border_soft, 5, 1)

                if gy >= body_y and gy < body_bottom then
                    draw.RectFilled(gx + 1, gy + 1, col_w - 2, header_h - 2, self.COLORS.panel_alt, 4)
                    draw.RectFilled(gx + 1, gy + 1, 3, header_h - 2, self.COLORS.accent, 2)
                    draw.Text(gx + 12, gy + 7, g.name, self.COLORS.text, 13)
                end

                local wy = gy + header_h + 2
                for _, w in ipairs(g.widgets) do
                    if self.state.visible[w.id] ~= false then
                        if wy >= body_y and wy + row_h <= body_bottom then
                            draw_widget(self, w, gx + 6, wy, col_w - 12, row_h, lmb_down, lmb_click, block_input)
                        end
                        wy = wy + row_h + 4
                    end
                end
            end
        end
        ys[ci] = gy + group_h + 10
    end

    if max_scroll > 0 then
        local sb_x = st.x + st.w - 6
        local sb_w = 4
        local bar_h = max(20, view_h * (view_h / total_h))
        local bar_y = body_y + (view_h - bar_h) * (st.scroll / max_scroll)
        draw.RectFilled(sb_x, body_y, sb_w, view_h, {1,1,1,0.05}, 2)
        draw.RectFilled(sb_x, bar_y, sb_w, bar_h, self.COLORS.accent, 2)
    end

    -- dropdown overlay SEMPRE por último
    draw_open_combo_overlay(self)

    if st.picker then
        local p = st.picker
        draw.RectFilled(p.x - 2, p.y - 2, p.w + 4, p.h + 4, {0,0,0,0.6}, 8)
        draw_color_picker(self, p.id, p.x, p.y, p.w, p.h)
        if lmb_click and not st.picker_just_opened and not picker_state.dragging_sv and not picker_state.dragging_hue then
            if not in_rect(mx, my, p.x - 10, p.y - 10, p.w + 20, p.h + 20) then
                st.picker = nil
                picker_state.hue = nil
            end
        end
    end

    st.combo_just_opened = false
    st.picker_just_opened = false
    st.prev_lmb = lmb_down
end

local function process_hotkey_listening(self)
    if not self.state.listening_key then return end
    if self.state.listen_wait_lmb_up then
        if not key_down(0x01) then self.state.listen_wait_lmb_up = false end
        return
    end
    if key_down(0x1B) then
        self.state.listening_key = nil
        return
    end
    for vk = 1, 254 do
        if vk ~= 0x01 and key_down(vk) then
            self.state.keys[self.state.listening_key] = vk
            self.state.listening_key = nil
            break
        end
    end
end

local _prev_toggle = false
local function process_toggle(self)
    local toggle_key = self.state.toggle_key
    if not toggle_key or toggle_key == 0 then
        _prev_toggle = false
        return
    end
    if self.state.listening_key == "ui_toggle_key" then
        _prev_toggle = key_down(toggle_key)
        return
    end
    local kd = key_down(toggle_key)
    if kd and not _prev_toggle then
        self.state.open = not self.state.open
    end
    _prev_toggle = kd
end

-- ═══════════════════════════════════════════════════════════════════════
-- WEAPONS
-- ═══════════════════════════════════════════════════════════════════════

local WEAPONS = {
    [1]  = { name = "SVD",               velocity = 1322.9, range = 1945.5, drop_mult = 1.13, patterns = { "svd" } },
    [2]  = { name = "Revolver",          velocity = 583.6,  range = 778.2,  drop_mult = 1.13, patterns = { "revolver" } },
    [3]  = { name = "MP5",               velocity = 680.9,  range = 389.1,  drop_mult = 1.13, patterns = { "mp5" } },
    [4]  = { name = "FNX-45",            velocity = 583.6,  range = 583.6,  drop_mult = 1.13, patterns = { "fnx" } },
    [5]  = { name = "Remington 700",     velocity = 1011.6, range = 1361.8, drop_mult = 1.13, patterns = { "hunting", "remington" } },
    [6]  = { name = "P226",              velocity = 583.6,  range = 389.1,  drop_mult = 1.13, patterns = { "p226" } },
    [7]  = { name = "MK4",               velocity = 583.6,  range = 466.9,  drop_mult = 1.13, patterns = { "mk4" } },
    [8]  = { name = "M1911",             velocity = 583.6,  range = 389.1,  drop_mult = 1.13, patterns = { "m1911" } },
    [9]  = { name = "M40A1",             velocity = 1011.6, range = 1361.8, drop_mult = 1.13, patterns = { "m40" } },
    [10] = { name = "Mossberg 500",      velocity = 389.1,  range = 101.1,  drop_mult = 1.13, patterns = { "mossberg" } },
    [11] = { name = "Recurve Bow",       velocity = 214.0,  range = 291.8,  drop_mult = 1.13, patterns = { "bow", "recurve" } },
    [12] = { name = "UMP-45",            velocity = 680.9,  range = 466.9,  drop_mult = 1.13, patterns = { "ump" } },
    [13] = { name = "UZI",               velocity = 680.9,  range = 389.1,  drop_mult = 1.13, patterns = { "uzi" } },
    [14] = { name = "AKM",               velocity = 875.4,  range = 972.7,  drop_mult = 1.13, patterns = { "akm", "ak47", "ak-47" } },
    [15] = { name = "AR-15",             velocity = 875.4,  range = 972.7,  drop_mult = 1.13, patterns = { "ar15", "ar-15" } },
    [16] = { name = "Glock",             velocity = 583.6,  range = 466.9,  drop_mult = 1.13, patterns = { "glock" } },
    [17] = { name = "M9",                velocity = 583.6,  range = 466.9,  drop_mult = 1.13, patterns = { "m9" } },
    [18] = { name = "FAMAS",             velocity = 875.4,  range = 972.7,  drop_mult = 1.13, patterns = { "famas" } },
    [19] = { name = "Sporter",           velocity = 680.9,  range = 700.3,  drop_mult = 1.13, patterns = { "sporter" } },
    [20] = { name = "Double Barrel",     velocity = 389.1,  range = 77.8,   drop_mult = 1.13, patterns = { "doublebarrel", "double barrel" } },
    [21] = { name = "Honey Badger",      velocity = 466.9,  range = 700.3,  drop_mult = 1.13, patterns = { "honeybadger", "honey" } },
    [22] = { name = "MCX",               velocity = 466.9,  range = 778.2,  drop_mult = 1.13, patterns = { "mcx" } },
    [23] = { name = "MK14 EBR",          velocity = 1322.9, range = 1945.5, drop_mult = 1.13, patterns = { "mk14" } },
    [24] = { name = "MK47 Mutant",       velocity = 875.4,  range = 972.7,  drop_mult = 1.13, patterns = { "mk47" } },
    [25] = { name = "TEC-9",             velocity = 583.6,  range = 466.9,  drop_mult = 1.13, patterns = { "tec9", "tec-9" } },
    [26] = { name = "PKM",               velocity = 875.4,  range = 1167.3, drop_mult = 1.13, patterns = { "pkm" } },
    [27] = { name = "T13 Crossbow",      velocity = 389.1,  range = 3891.0, drop_mult = 1.13, patterns = { "crossbow" } },
    [28] = { name = "Mosin Nagant",      velocity = 875.4,  range = 778.2,  drop_mult = 1.13, patterns = { "mosin" } },
    [29] = { name = "Desert Eagle",      velocity = 583.6,  range = 778.2,  drop_mult = 1.13, patterns = { "desert", "deagle" } },
    [30] = { name = "FN-FAL",            velocity = 875.4,  range = 1167.3, drop_mult = 1.13, patterns = { "fnfal", "fn-fal" } },
    [31] = { name = "M4A1",              velocity = 875.4,  range = 972.7,  drop_mult = 1.13, patterns = { "m4a1" } },
    [32] = { name = "SCAR-H",            velocity = 875.4,  range = 1167.3, drop_mult = 1.13, patterns = { "scar" } },
    [33] = { name = "MP-133",            velocity = 389.1,  range = 87.5,   drop_mult = 1.13, patterns = { "shotgun", "mp133" } },
    [34] = { name = "MRAD",              velocity = 1322.9, range = 1945.5, drop_mult = 1.13, patterns = { "mrad" } },
    [35] = { name = "Makarov",           velocity = 583.6,  range = 466.9,  drop_mult = 1.13, patterns = { "makarov", "makarov pistol" } },
    [36] = { name = "M249",              velocity = 875.4,  range = 1070.0, drop_mult = 1.13, patterns = { "m249", "m249 saw" } },
    [37] = { name = "Saiga-12",          velocity = 428.0,  range = 108.9,  drop_mult = 1.13, patterns = { "saiga" } },
    [38] = { name = "MK18",              velocity = 1322.9, range = 1945.5, drop_mult = 1.13, patterns = { "mk18" } },
    [39] = { name = "M110K",             velocity = 1322.9, range = 1945.5, drop_mult = 1.13, patterns = { "m110" } },
    [40] = { name = "SKS",               velocity = 875.4,  range = 972.7,  drop_mult = 1.13, patterns = { "sks" } },
    [41] = { name = "AWM",               velocity = 1322.9, range = 1945.5, drop_mult = 1.13, patterns = { "awm" } },
}

local WEAPON_COMBO = {}
for i, w in ipairs(WEAPONS) do WEAPON_COMBO[i] = w.name end
WEAPON_COMBO[#WEAPONS + 1] = "Auto (from weapon)"
local AUTO_INDEX = #WEAPONS + 1

local function combo_to_weapon(c) return (c or 0) + 1 end
local function weapon_to_combo(w) return (w or 1) - 1 end

local function clean_string(str)
    if not str then return "" end
    str = string.gsub(str, "%c", "")
    str = string.gsub(str, "^%s+", "")
    str = string.gsub(str, "%s+$", "")
    return str
end

local function find_weapon_index_by_name(name)
    if not name or name == "" then return nil end
    local lower_name = string.lower(clean_string(name))
    if lower_name == "" then return nil end
    for i, w in ipairs(WEAPONS) do
        if w.patterns then
            for _, pat in ipairs(w.patterns) do
                if string.find(lower_name, pat, 1, true) then return i end
            end
        end
    end
    return nil
end

local base_ui = setmetatable({}, UI)

-- ═══════════════════════════════════════════════════════════════════════
-- REGISTRO DA UI
-- ═══════════════════════════════════════════════════════════════════════

base_ui:add_tab("Visuals")
base_ui:add_tab("Aimbot")
base_ui:add_tab("Misc")
base_ui:add_tab("Config")

base_ui:add_group("Visuals", "ESP")
base_ui:add_checkbox("Visuals", "ESP", "esp_enabled", "Enable ESP", false)
base_ui:add_multicombo("Visuals", "ESP", "targets", "Targets", {"Players","Zombies"}, {true,false}, { parent = "esp_enabled" })
base_ui:add_checkbox("Visuals", "ESP", "disable_zombie_scan", "Disable Zombie Scan", false)
base_ui:add_checkbox("Visuals", "ESP", "only_special", "Only Special Zombies", false, { parent = "esp_enabled" })
base_ui:add_checkbox("Visuals", "ESP", "box", "Box", false, { parent = "esp_enabled", colorpicker = {1,1,1,1} })
base_ui:add_combo("Visuals", "ESP", "box_type", "Box Style", {"2D","Corner","3D"}, 0, { parent = "box" })
base_ui:add_checkbox("Visuals", "ESP", "fill", "Box Filled", false, { parent = "esp_enabled", colorpicker = {1,1,1,0.2} })
base_ui:add_slider_int("Visuals", "ESP", "fill_op", "Fill Opacity", 0, 100, 20, { parent = "fill" })
base_ui:add_checkbox("Visuals", "ESP", "name", "Name", false, { parent = "esp_enabled", colorpicker = {1,1,1,1} })
base_ui:add_checkbox("Visuals", "ESP", "distance", "Distance", false, { parent = "esp_enabled", colorpicker = {0.7,0.7,0.7,1} })
base_ui:add_checkbox("Visuals", "ESP", "skeleton", "Skeleton", false, { parent = "esp_enabled", colorpicker = {1,0.8,0.2,1} })
base_ui:add_checkbox("Visuals", "ESP", "head_dot", "Head Dot", false, { parent = "esp_enabled", colorpicker = {1,1,1,1} })
base_ui:add_checkbox("Visuals", "ESP", "viewline", "View Line", false, { parent = "esp_enabled", colorpicker = {1,1,1,1} })
base_ui:add_slider_int("Visuals", "ESP", "vl_length", "View Line Length", 1, 15, 5, { parent = "viewline" })
base_ui:add_combo("Visuals", "ESP", "vl_style", "View Line Style", {"Solid","Dashed","Fade"}, 2, { parent = "viewline" })
base_ui:add_checkbox("Visuals", "ESP", "snapline", "Snaplines", false, { parent = "esp_enabled", colorpicker = {1,1,1,0.5} })
base_ui:add_combo("Visuals", "ESP", "snap_mode", "Snapline Origin", {"Top","Center","Bottom"}, 2, { parent = "snapline" })
base_ui:add_slider_int("Visuals", "ESP", "max_distance", "Render Distance", 1, 5000, 5000, { parent = "esp_enabled" })
base_ui:add_slider_int("Visuals", "ESP", "font_size", "Text Size", 8, 24, 13, { parent = "esp_enabled" })

base_ui:add_group("Visuals", "Vehicles")
base_ui:add_checkbox("Visuals", "Vehicles", "veh_enabled", "Enable Vehicle ESP", false)
base_ui:add_checkbox("Visuals", "Vehicles", "veh_name", "Vehicle Name", true, { parent = "veh_enabled", colorpicker = {1,0.8,0.2,1} })
base_ui:add_checkbox("Visuals", "Vehicles", "veh_dist", "Vehicle Distance", true, { parent = "veh_enabled", colorpicker = {0.7,0.7,0.7,1} })
base_ui:add_checkbox("Visuals", "Vehicles", "veh_box", "Vehicle Box", false, { parent = "veh_enabled", colorpicker = {1,0.5,0,1} })
base_ui:add_slider_int("Visuals", "Vehicles", "veh_range", "Vehicle Range (m)", 10, 2000, 500, { parent = "veh_enabled" })
base_ui:add_slider_int("Visuals", "Vehicles", "veh_font", "Vehicle Font Size", 8, 24, 14, { parent = "veh_enabled" })

base_ui:add_group("Visuals", "Radar")
base_ui:add_checkbox("Visuals", "Radar", "radar_enabled", "Enable Radar", false)
base_ui:add_slider_int("Visuals", "Radar", "radar_size", "Radar Size", 80, 400, 300, { parent = "radar_enabled" })
base_ui:add_multicombo("Visuals", "Radar", "radar_targets", "Radar Targets", {"Players","Zombies"}, {true,true}, { parent = "radar_enabled" })
base_ui:add_slider_int("Visuals", "Radar", "radar_range", "Range (m)", 50, 2000, 200, { parent = "radar_enabled" })
base_ui:add_slider_int("Visuals", "Radar", "radar_dot", "Dot Size", 1, 8, 3, { parent = "radar_enabled" })

base_ui:add_group("Visuals", "Weapon HUD")
base_ui:add_checkbox("Visuals", "Weapon HUD", "wh_enabled", "Enable Weapon HUD", false)
base_ui:add_slider_int("Visuals", "Weapon HUD", "wh_x", "HUD X", 0, 2000, 20, { parent = "wh_enabled" })
base_ui:add_slider_int("Visuals", "Weapon HUD", "wh_y", "HUD Y", 0, 2000, 20, { parent = "wh_enabled" })
base_ui:add_slider_int("Visuals", "Weapon HUD", "wh_font", "Font Size", 8, 32, 16, { parent = "wh_enabled" })
base_ui:add_checkbox("Visuals", "Weapon HUD", "wh_debug", "Debug Overlay", false, { parent = "wh_enabled" })

base_ui:add_group("Aimbot", "Aimbot")
base_ui:add_checkbox("Aimbot", "Aimbot", "aim_enabled", "Enable Aimbot", false)
base_ui:add_checkbox("Aimbot", "Aimbot", "aim_zombies", "Zombie Aimbot", false, { parent = "aim_enabled" })
base_ui:add_hotkey("Aimbot", "Aimbot", "aim_key", "Aimbot Keybind", 0x02, { parent = "aim_enabled" })
base_ui:add_combo("Aimbot", "Aimbot", "aim_target_type", "Target Priority", {"Crosshair","Distance"}, 0, { parent = "aim_enabled" })
base_ui:add_slider_int("Aimbot", "Aimbot", "aim_smooth", "Smoothing (0 = Instant)", 0, 20, 5, { parent = "aim_enabled" })
base_ui:add_combo("Aimbot", "Aimbot", "aim_hitbox", "Hitbox", {"Head","Torso","Left Arm","Right Arm","Left Leg","Right Leg","Closest"}, 0, { parent = "aim_enabled" })
base_ui:add_checkbox("Aimbot", "Aimbot", "aim_predict", "Prediction (XYZ)", false, { parent = "aim_enabled" })
base_ui:add_checkbox("Aimbot", "Aimbot", "aim_fov_show", "Show FOV Circle", true, { parent = "aim_enabled", colorpicker = {1,1,1,1} })
base_ui:add_slider_int("Aimbot", "Aimbot", "aim_fov", "FOV", 1, 500, 120, { parent = "aim_enabled" })
base_ui:add_slider_int("Aimbot", "Aimbot", "aim_max_dist", "Max Distance (Player)", 1, 5000, 1000, { parent = "aim_enabled" })
base_ui:add_slider_int("Aimbot", "Aimbot", "aim_zombie_dist", "Max Distance (Zombie)", 1, 5000, 1000, { parent = "aim_zombies" })
base_ui:add_checkbox("Aimbot", "Aimbot", "aim_lock", "Target Lock", false, { parent = "aim_enabled" })
base_ui:add_checkbox("Aimbot", "Aimbot", "aim_line", "Target Line", false, { parent = "aim_enabled", colorpicker = {1,0,0,1} })
base_ui:add_combo("Aimbot", "Aimbot", "aim_line_style", "Target Line Style", {"Solid","Dashed","Dotted"}, 0, { parent = "aim_line" })

base_ui:add_group("Aimbot", "Ballistics")
base_ui:add_checkbox("Aimbot", "Ballistics", "bal_enabled", "Enable Ballistic Compensation", false)
base_ui:add_combo("Aimbot", "Ballistics", "bal_weapon", "Weapon Profile", WEAPON_COMBO, weapon_to_combo(AUTO_INDEX), { parent = "bal_enabled" })
base_ui:add_slider_int("Aimbot", "Ballistics", "bal_gravity_x100", "Gravity (m/s2 x100)", 100, 5000, floor(GAME_GRAVITY_MPS * 100), { parent = "bal_enabled" })
base_ui:add_slider_int("Aimbot", "Ballistics", "bal_zero_m", "Zero Range (m)", 0, 400, 0, { parent = "bal_enabled" })
base_ui:add_slider_int("Aimbot", "Ballistics", "bal_drop_mult_x100", "Drop Multiplier x100", 100, 200, 113, { parent = "bal_enabled" })
base_ui:add_checkbox("Aimbot", "Ballistics", "bal_show_marker", "Holdover Marker", true, { parent = "bal_enabled", colorpicker = {1,0.5,0,1} })
base_ui:add_checkbox("Aimbot", "Ballistics", "bal_show_text", "Distance / Drop Text", true, { parent = "bal_enabled" })

base_ui:add_group("Misc", "Desync Marker")
base_ui:add_checkbox("Misc", "Desync Marker", "ds_enabled", "Enable Desync Marker", false)
base_ui:add_hotkey("Misc", "Desync Marker", "ds_toggle_key", "Toggle Marker Key", 0x58, { parent = "ds_enabled" })
base_ui:add_hotkey("Misc", "Desync Marker", "ds_mode_key", "Toggle Camera Mode Key", 0x56, { parent = "ds_enabled" })
base_ui:add_combo("Misc", "Desync Marker", "ds_camera_mode", "Camera Mode", {"1st Person","3rd Person"}, 1, { parent = "ds_enabled" })
base_ui:add_checkbox("Misc", "Desync Marker", "ds_show_hud", "Show HUD", true, { parent = "ds_enabled" })
base_ui:add_checkbox("Misc", "Desync Marker", "ds_show_label", "Show Marker Label", true, { parent = "ds_enabled", colorpicker = {1,0,1,1} })
base_ui:add_checkbox("Misc", "Desync Marker", "ds_show_circle", "Show Desync Range Circle", true, { parent = "ds_enabled", colorpicker = {0.7,0.3,1,0.6} })
base_ui:add_checkbox("Misc", "Desync Marker", "ds_show_outside_warning", "Show Out-of-Range Warning", true, { parent = "ds_enabled" })

base_ui:add_group("Misc", "Info")
base_ui:add_label("Misc", "Info", "Press toggle key to open/close UI")
base_ui:add_label("Misc", "Info", "Drag titlebar to move")

base_ui:add_group("Config", "UI Customization")
base_ui:add_hotkey("Config", "UI Customization", "ui_toggle_key", "Menu Toggle Key", 0x2D)
base_ui:add_colorpicker("Config", "UI Customization", "ui_accent", "Accent Color", {0.25, 0.60, 1.00, 1.00})
base_ui:add_slider_int("Config", "UI Customization", "ui_opacity", "UI Opacity (%)", 30, 100, 96)
base_ui:add_slider_int("Config", "UI Customization", "ui_width", "Window Width", 500, 1600, 860)
base_ui:add_slider_int("Config", "UI Customization", "ui_height", "Window Height", 400, 1000, 640)

base_ui:set_callback("ui_toggle_key", function(k)
    base_ui.state.toggle_key = k
    _prev_toggle = key_down(k)
end)
base_ui:set_callback("ui_accent", function(c)
    base_ui.state.ui_settings.accent = c
    base_ui:apply_ui_settings()
end)
base_ui:set_callback("ui_opacity", function(v)
    base_ui.state.ui_settings.opacity = v
    base_ui:apply_ui_settings()
end)
base_ui:set_callback("ui_width", function(v)
    base_ui.state.ui_settings.window_w = v
    base_ui:apply_ui_settings()
end)
base_ui:set_callback("ui_height", function(v)
    base_ui.state.ui_settings.window_h = v
    base_ui:apply_ui_settings()
end)

base_ui:add_group("Config", "Config")

-- ═══════════════════════════════════════════════════════════════════════
-- SETTINGS SYNC / VISIBILITY
-- ═══════════════════════════════════════════════════════════════════════

local s = {}
local function sync_settings()
    for id, v in pairs(base_ui.state.values) do s[id] = v end
    for id, c in pairs(base_ui.state.colors) do s[id .. "_color"] = c end
    for id, k in pairs(base_ui.state.keys) do s[id .. "_key"] = k end
end

local function update_visibility()
    local master = s.esp_enabled == true
    local dzs = s.disable_zombie_scan == true

    base_ui:set_visible("targets", master)
    base_ui:set_visible("box_type", master and s.box == true)
    base_ui:set_visible("fill_op", master and s.fill == true)
    base_ui:set_visible("vl_length", master and s.viewline == true)
    base_ui:set_visible("vl_style", master and s.viewline == true)
    base_ui:set_visible("snap_mode", master and s.snapline == true)

    local veh = s.veh_enabled == true
    base_ui:set_visible("veh_name", veh)
    base_ui:set_visible("veh_dist", veh)
    base_ui:set_visible("veh_box", veh)
    base_ui:set_visible("veh_range", veh)
    base_ui:set_visible("veh_font", veh)

    local am = s.aim_enabled == true
    base_ui:set_visible("aim_key", am)
    base_ui:set_visible("aim_zombies", am and not dzs)
    base_ui:set_visible("aim_target_type", am)
    base_ui:set_visible("aim_smooth", am)
    base_ui:set_visible("aim_hitbox", am)
    base_ui:set_visible("aim_fov_show", am)
    base_ui:set_visible("aim_fov", am)
    base_ui:set_visible("aim_max_dist", am)
    base_ui:set_visible("aim_zombie_dist", am and s.aim_zombies == true and not dzs)
    base_ui:set_visible("aim_lock", am)
    base_ui:set_visible("aim_line", am)
    base_ui:set_visible("aim_line_style", am and s.aim_line == true)
    base_ui:set_visible("aim_predict", am)

    local bal = s.bal_enabled == true
    base_ui:set_visible("bal_weapon", bal)
    base_ui:set_visible("bal_gravity_x100", bal)
    base_ui:set_visible("bal_zero_m", bal)
    base_ui:set_visible("bal_drop_mult_x100", bal)
    base_ui:set_visible("bal_show_marker", bal)
    base_ui:set_visible("bal_show_text", bal)

    local rm = s.radar_enabled == true
    base_ui:set_visible("radar_size", rm)
    base_ui:set_visible("radar_targets", rm)
    base_ui:set_visible("radar_range", rm)
    base_ui:set_visible("radar_dot", rm)

    local wh = s.wh_enabled == true
    base_ui:set_visible("wh_x", wh)
    base_ui:set_visible("wh_y", wh)
    base_ui:set_visible("wh_font", wh)
    base_ui:set_visible("wh_debug", wh)

    local ds = s.ds_enabled == true
    base_ui:set_visible("ds_toggle_key", ds)
    base_ui:set_visible("ds_mode_key", ds)
    base_ui:set_visible("ds_camera_mode", ds)
    base_ui:set_visible("ds_show_hud", ds)
    base_ui:set_visible("ds_show_label", ds)
    base_ui:set_visible("ds_show_circle", ds)
    base_ui:set_visible("ds_show_outside_warning", ds)
end

-- ═══════════════════════════════════════════════════════════════════════
-- HELPERS DO AFTERMATH
-- ═══════════════════════════════════════════════════════════════════════

local cached_cw = nil
local function find_current_weapon()
    if cached_cw and cached_cw.Parent then return cached_cw end
    cached_cw = nil
    for _, obj in ipairs(game.Workspace:GetDescendants()) do
        if obj.Name == "CurrentWeapon" and obj:IsA("Model") then
            cached_cw = obj
            return obj
        end
    end
    return nil
end

local function get_local_weapon()
    local cw = find_current_weapon()
    if cw then
        local pointer = cw:FindFirstChild("Pointer")
        if pointer and pointer:IsA("ObjectValue") and pointer.Value then
            local name = pointer.Value.Name
            if name and name ~= "" then return name, "Pointer" end
        end
        local item = cw:FindFirstChild("InventoryItem")
        if item and item:IsA("ObjectValue") and item.Value then
            return item.Value.Name, "InventoryItem"
        end
    end
    return nil, "not found"
end

local function get_active_weapon()
    local auto_combo = weapon_to_combo(AUTO_INDEX)
    local combo_val = tonumber(s.bal_weapon) or auto_combo
    local weapon_idx = combo_to_weapon(combo_val)
    local weapon = WEAPONS[weapon_idx]
    if not weapon then
        local weapon_name = get_local_weapon()
        local matched = find_weapon_index_by_name(weapon_name)
        if matched then weapon = WEAPONS[matched] end
    end
    return weapon
end

local bal_auto_mode = true
local bal_last_written = nil
local bal_last_effective = nil

local function autoswitch_ballistics()
    local auto_combo = weapon_to_combo(AUTO_INDEX)
    local current = tonumber(s.bal_weapon) or auto_combo
    if bal_last_written == nil then
        bal_last_written = current
    elseif current ~= bal_last_written then
        bal_auto_mode = (current == auto_combo)
        bal_last_written = current
    end
    if not bal_auto_mode then
        local w_idx = combo_to_weapon(current)
        local w = WEAPONS[w_idx]
        if w and bal_last_effective ~= w_idx then
            bal_last_effective = w_idx
            base_ui:set("bal_drop_mult_x100", floor((w.drop_mult or 1.13) * 100))
        end
        return
    end
    local weapon_name = get_local_weapon()
    local matched = find_weapon_index_by_name(weapon_name)
    if matched and matched ~= bal_last_effective then
        bal_last_effective = matched
        local combo = weapon_to_combo(matched)
        bal_last_written = combo
        base_ui:set("bal_weapon", combo)
        local w = WEAPONS[matched]
        if w then
            base_ui:set("bal_drop_mult_x100", floor((w.drop_mult or 1.13) * 100))
        end
    end
end

-- ═══════════════════════════════════════════════════════════════════════
-- VEHICLES
-- ═══════════════════════════════════════════════════════════════════════

local VEHICLES = {}
local veh_last_scan = 0
local VEH_SCAN_INTERVAL_MS = 800

local function is_vehicle_model(model)
    if not model or model.ClassName ~= "Model" then return false end
    if model.Parent ~= game.Workspace then return false end
    if model.Name ~= "WorldModel" then return false end
    local mesh_count = 0
    local has_wheel = false
    for _, c in ipairs(model:GetDescendants()) do
        if c:IsA("MeshPart") then
            mesh_count = mesh_count + 1
            if not has_wheel then
                local lname = string.lower(c.Name)
                if string.find(lname, "wheel", 1, true) then has_wheel = true end
            end
        end
    end
    return mesh_count >= 6 and has_wheel
end

local function get_vehicle_root(model)
    local best, best_size = nil, 0
    for _, c in ipairs(model:GetChildren()) do
        if c:IsA("BasePart") then
            local sz = c.Size
            local vol = sz.X * sz.Y * sz.Z
            if vol > best_size then
                best_size = vol
                best = c
            end
        end
    end
    return best
end

local function get_vehicle_display_name(model)
    local has_rotator = false
    local has_truck_wheel = false
    local has_wheel1 = false
    for _, c in ipairs(model:GetDescendants()) do
        if c:IsA("MeshPart") then
            local lname = string.lower(c.Name)
            if string.find(lname, "rotator", 1, true) then has_rotator = true end
            if string.find(lname, "truck_wheel", 1, true) then has_truck_wheel = true end
            if string.find(lname, "wheel1_mesh", 1, true) then has_wheel1 = true end
        end
    end
    if has_rotator or has_truck_wheel then return "Police Car" end
    if has_wheel1 then return "Pickup" end
    for _, c in ipairs(model:GetChildren()) do
        if c:IsA("MeshPart") then
            local lname = string.lower(c.Name)
            if not string.find(lname, "wheel", 1, true)
                and not string.find(lname, "rotator", 1, true)
                and not string.find(lname, "collision", 1, true)
                and not string.find(lname, "stand", 1, true) then
                return c.Name
            end
        end
    end
    return "Vehicle"
end

local function scan_vehicles()
    local now = utility.GetTickCount()
    if now - veh_last_scan < VEH_SCAN_INTERVAL_MS then return end
    veh_last_scan = now
    local result = {}
    for _, child in ipairs(game.Workspace:GetChildren()) do
        if is_vehicle_model(child) then
            local root = get_vehicle_root(child)
            if root then
                result[#result + 1] = {
                    model = child,
                    root = root,
                    name = get_vehicle_display_name(child),
                }
            end
        end
    end
    VEHICLES = result
end

local function draw_vehicles(cam)
    if not s.veh_enabled then return end
    if #VEHICLES == 0 then return end

    local fs = tonumber(s.veh_font) or 14
    local range_m = tonumber(s.veh_range) or 500
    local range_studs = range_m * STUDS_PER_METER
    local range_sq = range_studs * range_studs

    local show_name = s.veh_name
    local show_dist = s.veh_dist
    local show_box = s.veh_box
    local name_col = s.veh_name_color or {1, 0.8, 0.2, 1}
    local dist_col = s.veh_dist_color or {0.7, 0.7, 0.7, 1}
    local box_col = s.veh_box_color or {1, 0.5, 0, 1}

    local cam_x, cam_y, cam_z = cam.X, cam.Y, cam.Z

    for i = 1, #VEHICLES do
        local v = VEHICLES[i]
        local root = v.root
        if root and root.Parent then
            local pos = root.Position
            local dx, dy, dz = pos.X-cam_x, pos.Y-cam_y, pos.Z-cam_z
            local dist_sq = dx*dx + dy*dy + dz*dz
            if dist_sq <= range_sq then
                local sx, sy, on_screen = draw.WorldToScreen(pos.X, pos.Y, pos.Z)
                if on_screen then
                    local dist_studs = sqrt(dist_sq)
                    local dist_m = floor(dist_studs / STUDS_PER_METER)
                    if show_box then
                        local mnx, mny, mxx, mxy = 1e9, 1e9, -1e9, -1e9
                        local any = false
                        local sz = root.Size
                        local cf = root.CFrame
                        for ox = -1, 1, 2 do
                            for oy = -1, 1, 2 do
                                for oz = -1, 1, 2 do
                                    local world_pos = cf * Vector3.New(
                                        sz.X * 0.5 * ox,
                                        sz.Y * 0.5 * oy,
                                        sz.Z * 0.5 * oz
                                    )
                                    local px, py, pv = draw.WorldToScreen(world_pos.X, world_pos.Y, world_pos.Z)
                                    if pv then
                                        any = true
                                        if px < mnx then mnx = px end
                                        if px > mxx then mxx = px end
                                        if py < mny then mny = py end
                                        if py > mxy then mxy = py end
                                    end
                                end
                            end
                        end
                        if any then
                            draw.Rect(mnx, mny, mxx - mnx, mxy - mny, box_col, 0, 1.5)
                        end
                    end
                    local label_parts = {}
                    if show_name then label_parts[#label_parts + 1] = v.name end
                    if show_dist then label_parts[#label_parts + 1] = dist_m .. "m" end
                    if #label_parts > 0 then
                        local txt = table.concat(label_parts, "  ")
                        local tw = draw.GetTextSize(txt, fs)
                        local col = show_name and name_col or dist_col
                        draw.Text(sx - tw * 0.5, sy - 12, txt, col, fs)
                    end
                end
            end
        end
    end
end

-- ═══════════════════════════════════════════════════════════════════════
-- ENTITIES
-- ═══════════════════════════════════════════════════════════════════════

local PART_CLASS = {Part = true, MeshPart = true, WedgePart = true}

local SKELETON = {
    R6 = {
        {"Head", "Torso"}, {"Torso", "Left Arm"}, {"Torso", "Right Arm"},
        {"Torso", "Left Leg"}, {"Torso", "Right Leg"}
    },
    R15 = {
        {"Head", "UpperTorso"},
        {"UpperTorso", "LowerTorso"},
        {"UpperTorso", "LeftUpperArm"}, {"LeftUpperArm", "LeftLowerArm"},
        {"UpperTorso", "RightUpperArm"}, {"RightUpperArm", "RightLowerArm"},
        {"LowerTorso", "LeftUpperLeg"}, {"LeftUpperLeg", "LeftLowerLeg"},
        {"LowerTorso", "RightUpperLeg"}, {"RightUpperLeg", "RightLowerLeg"}
    }
}

local BODY_PARTS = {
    R6 = {"Head", "Torso", "Left Arm", "Right Arm", "Left Leg", "Right Leg"},
    R15 = {
        "Head", "UpperTorso", "LowerTorso",
        "LeftUpperArm", "LeftLowerArm", "LeftHand",
        "RightUpperArm", "RightLowerArm", "RightHand",
        "LeftUpperLeg", "LeftLowerLeg", "LeftFoot",
        "RightUpperLeg", "RightLowerLeg", "RightFoot"
    }
}

local HITBOX_PART = {
    R6 = {[0] = "Head", [1] = "Torso", [2] = "Left Arm", [3] = "Right Arm", [4] = "Left Leg", [5] = "Right Leg"},
    R15 = {[0] = "Head", [1] = "UpperTorso", [2] = "LeftUpperArm", [3] = "RightUpperArm", [4] = "LeftUpperLeg", [5] = "RightUpperLeg"}
}

local aim_locked = nil
local aim_target = nil
local bal_marker = nil

local entries = {}
local entry_count = 0

local VELOCITY = {}
local VEL_SAMPLES = 3
local STILL_SNAP_SQ = 0.5

local function track_velocity_for(hrp)
    local addr = hrp.Address
    if not addr then return end
    local pos = hrp.Position
    if not pos then return end
    local state = VELOCITY[addr]
    if not state then
        VELOCITY[addr] = {
            vel = Vector3.New(0, 0, 0),
            prev = pos,
            prev_t = (utility.GetTime and utility.GetTime()) or os.clock(),
            samples = {}
        }
        return
    end
    local now = (utility.GetTime and utility.GetTime()) or os.clock()
    local dt = now - state.prev_t
    if dt > 0.0001 and dt < 0.25 then
        local dx = pos.X - state.prev.X
        local dy = pos.Y - state.prev.Y
        local dz = pos.Z - state.prev.Z
        local inst = Vector3.New(dx / dt, dy / dt, dz / dt)
        state.samples[#state.samples + 1] = inst
        if #state.samples > VEL_SAMPLES then
            table.remove(state.samples, 1)
        end
        local sx, sy, sz = 0, 0, 0
        local n = #state.samples
        for k = 1, n do
            local v = state.samples[k]
            sx = sx + v.X
            sy = sy + v.Y
            sz = sz + v.Z
        end
        local avg = Vector3.New(sx / n, sy / n, sz / n)
        local mag_xz_sq = avg.X*avg.X + avg.Z*avg.Z
        local mag_y = abs(avg.Y)
        if mag_xz_sq < STILL_SNAP_SQ and mag_y < 1.0 then
            state.vel = Vector3.New(0, 0, 0)
        else
            state.vel = avg
        end
    end
    state.prev = pos
    state.prev_t = now
end

thread.Create(function()
    local ga = game.Workspace and game.Workspace:FindFirstChild("game_assets")
    local folder = ga and ga:FindFirstChild("Entities")
    if not folder then return end
    local live = {}
    for _, m in ipairs(folder:GetChildren()) do
        if m.ClassName == "Model" then
            local hrp = m:FindFirstChild("HumanoidRootPart")
            if hrp and hrp.Address then
                live[hrp.Address] = true
                track_velocity_for(hrp)
            end
        end
    end
    for addr in pairs(VELOCITY) do
        if not live[addr] then VELOCITY[addr] = nil end
    end
end, 16)

local local_addr = nil
local LOCAL_ACQUIRE_SQ = 12 * 12

local function is_local_entity(e)
    local hrp = e and e.hrp
    return hrp ~= nil and hrp.Address ~= nil and hrp.Address == local_addr
end

local function update_local_addr(cam)
    if not cam then return end
    if local_addr then
        for i = 1, entry_count do
            local h = entries[i].hrp
            if h and h.Address == local_addr and utility.IsValid(h) then return end
        end
        local_addr = nil
    end
    local cx, cy, cz = cam.X, cam.Y, cam.Z
    local best, bd = nil, LOCAL_ACQUIRE_SQ
    for i = 1, entry_count do
        local h = entries[i].hrp
        if h and utility.IsValid(h) then
            local p = h.Position
            if p then
                local dx, dy, dz = p.X - cx, p.Y - cy, p.Z - cz
                local d = dx * dx + dy * dy + dz * dz
                if d <= bd then bd, best = d, h.Address end
            end
        end
    end
    if best then local_addr = best end
end

local function classify(model)
    local hrp = model:FindFirstChild("HumanoidRootPart")
    if not hrp then return nil end
    local kind, rig
    local is_special = false
    if model:FindFirstChild("UpperTorso") then
        kind, rig = "player", "R15"
    elseif model:FindFirstChild("Torso") then
        if s.disable_zombie_scan then return nil end
        kind, rig = "zombie", "R6"
        local equip = model:FindFirstChild("Equipment")
        if equip then
            local eq_model = equip:FindFirstChild("Model")
            if eq_model then
                local eq_head = eq_model:FindFirstChild("Head")
                if eq_head then
                    local eq_main = eq_head:FindFirstChild("Main")
                    if eq_main then
                        if eq_main:FindFirstChild("AttachmentHelmetMask") then
                            is_special = true
                        end
                    end
                end
            end
        end
    else
        return nil
    end
    local body = {}
    for _, name in ipairs(BODY_PARTS[rig]) do
        local p = model:FindFirstChild(name)
        if p and PART_CLASS[p.ClassName] then body[#body + 1] = p end
    end
    local skel = {}
    for _, pair in ipairs(SKELETON[rig]) do
        local a = model:FindFirstChild(pair[1])
        local b = model:FindFirstChild(pair[2])
        if a and b then skel[#skel + 1] = {a, b} end
    end
    return {
        kind = kind, rig = rig, hrp = hrp,
        head = model:FindFirstChild("Head"),
        model = model, body = body, skel = skel,
        is_special = is_special
    }
end

local SCAN_BUDGET = 12
local scan_queue = nil
local scan_index = 1
local scan_build = nil
local last_disable_zombie = false

local function rescan_step()
    if last_disable_zombie ~= s.disable_zombie_scan then
        last_disable_zombie = s.disable_zombie_scan
        scan_queue = nil
        scan_build = nil
        scan_index = 1
    end
    if not scan_queue then
        local ga = game.Workspace and game.Workspace:FindFirstChild("game_assets")
        local folder = ga and ga:FindFirstChild("Entities")
        if not folder then
            entries = {}
            entry_count = 0
            return
        end
        scan_queue = folder:GetChildren()
        scan_index = 1
        scan_build = {}
    end
    local last = min(scan_index + SCAN_BUDGET - 1, #scan_queue)
    for i = scan_index, last do
        local model = scan_queue[i]
        if model and model.ClassName == "Model" then
            local e = classify(model)
            if e then scan_build[#scan_build + 1] = e end
        end
    end
    scan_index = last + 1
    if scan_index > #scan_queue then
        entries = scan_build
        entry_count = #entries
        scan_queue = nil
        scan_build = nil
    end
end

thread.Create(function() pcall(rescan_step) end, 80)

local sw, sh = 0, 0

local function styled_line(x1, y1, x2, y2, col, style)
    if style == LINE_STYLE.DASHED then
        for i = 0, 4 do
            local t1, t2 = i / 5, (i + 0.5) / 5
            draw.Line(x1 + (x2-x1)*t1, y1 + (y2-y1)*t1,
                      x1 + (x2-x1)*t2, y1 + (y2-y1)*t2, col, 2)
        end
    elseif style == LINE_STYLE.FADE then
        for i = 0, 11 do
            local t1, t2 = i / 12, (i + 1) / 12
            local a = (col[4] or 1) * (1 - t1)
            draw.Line(x1 + (x2-x1)*t1, y1 + (y2-y1)*t1,
                      x1 + (x2-x1)*t2, y1 + (y2-y1)*t2,
                      {col[1], col[2], col[3], a}, 2)
        end
    else
        draw.Line(x1, y1, x2, y2, col, 2)
    end
end

local HAVE_NATIVE_CORNER = type(draw.CornerBox) == "function"

local function corner_box(x, y, w, h, col)
    if HAVE_NATIVE_CORNER then
        draw.CornerBox(x, y, w, h, col)
        return
    end
    local cl = min(w, h) * 0.25
    draw.Line(x, y, x+cl, y, col, 2)
    draw.Line(x, y, x, y+cl, col, 2)
    draw.Line(x+w-cl, y, x+w, y, col, 2)
    draw.Line(x+w, y, x+w, y+cl, col, 2)
    draw.Line(x, y+h-cl, x, y+h, col, 2)
    draw.Line(x, y+h, x+cl, y+h, col, 2)
    draw.Line(x+w-cl, y+h, x+w, y+h, col, 2)
    draw.Line(x+w, y+h-cl, x+w, y+h, col, 2)
end

local function draw_3d_box(bb, col)
    local c = {
        {bb[1], bb[2], bb[3]}, {bb[4], bb[2], bb[3]},
        {bb[4], bb[5], bb[3]}, {bb[1], bb[5], bb[3]},
        {bb[1], bb[2], bb[6]}, {bb[4], bb[2], bb[6]},
        {bb[4], bb[5], bb[6]}, {bb[1], bb[5], bb[6]}
    }
    local sp = {}
    for i = 1, 8 do
        local x, y, v = draw.WorldToScreen(c[i][1], c[i][2], c[i][3])
        sp[i] = {x, y, v}
    end
    local edges = {
        {1,2},{2,3},{3,4},{4,1},{5,6},{6,7},{7,8},{8,5},
        {1,5},{2,6},{3,7},{4,8}
    }
    for _, e in ipairs(edges) do
        local a, b = sp[e[1]], sp[e[2]]
        if a[3] and b[3] then
            draw.Line(a[1], a[2], b[1], b[2], col, 1.5)
        end
    end
end

local function project(e, cam)
    local hp = e.hrp.Position
    if not hp then return nil end
    local dx, dy, dz = hp.X-cam.X, hp.Y-cam.Y, hp.Z-cam.Z
    local dist_sq = dx*dx + dy*dy + dz*dz
    local dist = sqrt(dist_sq)

    local mnx, mny, mxx, mxy, any = 1e9, 1e9, -1e9, -1e9, false
    local wnx, wny, wnz = 1e9, 1e9, 1e9
    local wxx, wxy, wxz = -1e9, -1e9, -1e9
    for i = 1, #e.body do
        local part = e.body[i]
        local pos, sz = part.Position, part.Size
        if pos then
            local px, py, pv = draw.WorldToScreen(pos.X, pos.Y, pos.Z)
            if pv then
                any = true
                mnx, mny = min(mnx, px), min(mny, py)
                mxx, mxy = max(mxx, px), max(mxy, py)
            end
            local hx = sz and sz.X * 0.5 or 0
            local hy = sz and sz.Y * 0.5 or 0
            local hz = sz and sz.Z * 0.5 or 0
            wnx, wny, wnz = min(wnx, pos.X-hx), min(wny, pos.Y-hy), min(wnz, pos.Z-hz)
            wxx, wxy, wxz = max(wxx, pos.X+hx), max(wxy, pos.Y+hy), max(wxz, pos.Z+hz)
        end
    end
    if not any then return nil end
    local padx = (mxx - mnx) * 0.25 + 3
    local pady = (mxy - mny) * 0.12 + 6
    local b = {x = mnx - padx, y = mny - pady, w = (mxx-mnx) + padx*2, h = (mxy-mny) + pady*2}
    local bb = {wnx, wny, wnz, wxx, wxy, wxz}
    return b, dist, bb
end

local function render_entity(e, cam)
    local b, dist, bb = project(e, cam)
    if not b or dist > s.max_distance then return end

    local col
    if e.kind == "player" then
        col = s.box_color or {1, 1, 1, 1}
    elseif e.is_special then
        col = {1, 0.8, 0, 1}
    else
        col = s.box_color or {1, 1, 1, 1}
    end

    local cx = b.x + b.w * 0.5
    local btype = s.box_type or 0
    local too_small = b.h < 12

    if s.box and b.h > 6 then
        if s.fill then
            local fc = s.fill_color or {0, 0, 0, 0.4}
            draw.RectFilled(b.x, b.y, b.w, b.h,
                {fc[1], fc[2], fc[3], (s.fill_op or 20) * 0.01})
        end
        if btype == BOX_TYPE.CORNER then
            corner_box(b.x, b.y, b.w, b.h, col)
        elseif btype == BOX_TYPE.THREE_D then
            draw_3d_box(bb, col)
        else
            draw.Rect(b.x, b.y, b.w, b.h, col, 0, 1.5)
        end
    end

    if s.name then
        local label
        if e.kind == "player" then label = "Player"
        elseif e.is_special then label = "SPECIAL ZOMBIE"
        else label = "Zombie" end
        local nc
        if e.kind == "player" then nc = s.name_color or {1, 1, 1, 1}
        elseif e.is_special then nc = {1, 0.8, 0, 1}
        else nc = {1, 0.4, 0.4, 1} end
        local fs = s.font_size or 13
        local tw = draw.GetTextSize(label, fs)
        draw.Text(cx - tw * 0.5, b.y - fs - 2, label, nc, fs)
    end

    if s.skeleton and not too_small then
        local sc = e.is_special and {1, 0.8, 0, 1} or (s.skeleton_color or col)
        for k = 1, #e.skel do
            local ap, bp = e.skel[k][1].Position, e.skel[k][2].Position
            if ap and bp then
                local ax, ay, av = draw.WorldToScreen(ap.X, ap.Y, ap.Z)
                local bx, by, bv = draw.WorldToScreen(bp.X, bp.Y, bp.Z)
                if av and bv then draw.Line(ax, ay, bx, by, sc, 2) end
            end
        end
    end

    if s.head_dot and not too_small and e.head and e.head.Position then
        local hpos = e.head.Position
        local hx, hy, hv = draw.WorldToScreen(hpos.X, hpos.Y, hpos.Z)
        if hv then draw.Circle(hx, hy, 4, e.is_special and {1, 0.8, 0, 1} or (s.head_dot_color or col), 16, 1.5) end
    end

    if s.viewline then
        local lv = e.hrp.LookVector or e.hrp.look_vector
        local origin = (e.head and e.head.Position) or e.hrp.Position
        if lv and origin then
            local len = s.vl_length or 5
            local ex, ey, ez = origin.X + lv.X*len, origin.Y + lv.Y*len, origin.Z + lv.Z*len
            local sx, sy, v1 = draw.WorldToScreen(origin.X, origin.Y, origin.Z)
            local ex2, ey2, v2 = draw.WorldToScreen(ex, ey, ez)
            if v1 and v2 then
                styled_line(sx, sy, ex2, ey2, s.viewline_color or col, s.vl_style or 0)
            end
        end
    end

    if s.snapline then
        local mode = s.snap_mode or SNAP_MODE.BOTTOM
        local snap_y = mode == SNAP_MODE.TOP and 0 or mode == SNAP_MODE.CENTER and sh * 0.5 or sh
        draw.Line(sw * 0.5, snap_y, cx, b.y + b.h, s.snapline_color or col, 1.5)
    end

    if s.distance then
        local fs = s.font_size or 13
        local meters = dist / STUDS_PER_METER
        local txt = floor(meters) .. "m"
        local tw = draw.GetTextSize(txt, fs)
        draw.Text(cx - tw * 0.5, b.y + b.h + 2, txt, s.distance_color or {0.7, 0.7, 0.7, 1}, fs)
    end
end

-- ═══════════════════════════════════════════════════════════════════════
-- AIMBOT
-- ═══════════════════════════════════════════════════════════════════════

local function fov_center()
    local mx, my
    if utility.GetMousePos then
        mx, my = utility.GetMousePos()
    elseif input.GetMousePosition then
        mx, my = input.GetMousePosition()
    end
    if mx and my and (mx ~= 0 or my ~= 0) then
        return mx, my
    end
    return sw * 0.5, sh * 0.5
end

local function compute_flight_time(e)
    local weapon = get_active_weapon()
    if not weapon then return 0 end
    local hrp = e.hrp
    local target_pos = hrp and hrp.Position
    local cam_pos = camera.GetPosition()
    if not target_pos or not cam_pos then return 0 end
    local dist_studs = (target_pos - cam_pos).Magnitude
    local meters = dist_studs / STUDS_PER_METER
    local vel_mps = weapon.velocity or 1322.9
    local mult = tonumber(s.bal_drop_mult_x100)
    if mult then mult = mult / 100 else mult = (weapon.drop_mult or 1.13) end
    if mult <= 0 then mult = 1.13 end
    return (meters / vel_mps) * mult
end

local function apply_prediction(base, e)
    if not base then return nil end
    if not s.aim_predict then return base end
    local hrp = e.hrp
    local addr = hrp and hrp.Address
    if not addr then return base end
    local state = VELOCITY[addr]
    if not state then return base end
    local vel = state.vel
    local mag_xz_sq = vel.X*vel.X + vel.Z*vel.Z
    local mag_y = abs(vel.Y)
    if mag_xz_sq < 0.05 and mag_y < 0.5 then return base end
    local flight_time = compute_flight_time(e)
    if flight_time <= 0 then return base end
    if flight_time > 2.0 then flight_time = 2.0 end
    return Vector3.New(
        base.X + vel.X * flight_time,
        base.Y + vel.Y * flight_time,
        base.Z + vel.Z * flight_time
    )
end

local function apply_ballistics(base, e)
    bal_marker = nil
    if not s.bal_enabled then return base end
    if not base then return base end
    local weapon = get_active_weapon()
    if not weapon then return base end
    local hrp = e.hrp
    local target_pos = hrp and hrp.Position
    local cam_pos = camera.GetPosition()
    if not target_pos or not cam_pos then return base end
    local dist_studs = (target_pos - cam_pos).Magnitude
    if dist_studs < 1 then return base end
    local meters = dist_studs / STUDS_PER_METER
    local zero_m = tonumber(s.bal_zero_m) or 0
    if zero_m < 0 then zero_m = 0 end
    if zero_m > meters * 0.9 then zero_m = 0 end
    local effective_m = meters - zero_m
    if effective_m <= 0 then return base end
    local vel_mps = weapon.velocity
    if not vel_mps or vel_mps <= 0 then vel_mps = 1322.9 end
    local g = (tonumber(s.bal_gravity_x100) or 0) / 100
    if g <= 0 then g = GAME_GRAVITY_MPS end
    local t = effective_m / vel_mps
    local drop_m = 0.5 * g * t * t
    local mult = (tonumber(s.bal_drop_mult_x100) or 0) / 100
    if mult <= 0 then mult = (weapon.drop_mult or 1.13) end
    if mult <= 0 then mult = 1.13 end
    weapon.drop_mult = mult
    drop_m = drop_m * mult
    local y_world = drop_m * STUDS_PER_METER
    local bx, by, bv = draw.WorldToScreen(base.X, base.Y + y_world, base.Z)
    if bv then
        bal_marker = {
            sx = bx, sy = by,
            meters = meters, drop_m = drop_m, y_world = y_world,
        }
    end
    return Vector3.New(base.X, base.Y + y_world, base.Z)
end

local function hitbox_pos(e, cam)
    local hb = s.aim_hitbox or 0
    local base
    if hb == 6 then
        local cxp, cyp = fov_center()
        local best, bd
        for i = 1, #e.body do
            local p = e.body[i].Position
            if p then
                local px, py, v = draw.WorldToScreen(p.X, p.Y, p.Z)
                if v then
                    local d = (px-cxp)^2 + (py-cyp)^2
                    if not bd or d < bd then bd, best = d, p end
                end
            end
        end
        if not best then best = (e.head and e.head.Position) or e.hrp.Position end
        base = best
    else
        local name = HITBOX_PART[e.rig] and HITBOX_PART[e.rig][hb]
        local part = name and (e.hrp.Parent and e.hrp.Parent:FindFirstChild(name))
        base = (part and part.Position) or (e.head and e.head.Position) or e.hrp.Position
    end
    base = apply_prediction(base, e)
    if base then
        base = apply_ballistics(base, e)
    end
    return base
end

local function aim_candidate(e, cam)
    if is_local_entity(e) then return false end
    local p = e.hrp.Position
    if not p then return false end
    local dx, dy, dz = p.X-cam.X, p.Y-cam.Y, p.Z-cam.Z
    local dsq = dx*dx + dy*dy + dz*dz
    if e.kind == "player" then
        return dsq <= (s.aim_max_dist or 1000)^2
    elseif e.kind == "zombie" then
        return s.aim_zombies and dsq <= (s.aim_zombie_dist or 1000)^2
    end
    return false
end

local function do_aimbot(cam)
    aim_target = nil
    bal_marker = nil
    if not s.aim_enabled then
        aim_locked = nil
        return
    end
    local key = s.aim_key or 0x02
    if key <= 0 or not input.IsKeyDown(key) then
        aim_locked = nil
        return
    end
    local cxp, cyp = fov_center()
    local fov = s.aim_fov or 120
    local chosen
    if s.aim_lock and aim_locked and utility.IsValid(aim_locked.hrp)
        and aim_candidate(aim_locked, cam) then
        chosen = aim_locked
    else
        local best, bestscore
        for i = 1, entry_count do
            local e = entries[i]
            if utility.IsValid(e.hrp) and aim_candidate(e, cam) then
                local hp = hitbox_pos(e, cam)
                if hp then
                    local sx, sy, v = draw.WorldToScreen(hp.X, hp.Y, hp.Z)
                    if v then
                        local d = sqrt((sx-cxp)^2 + (sy-cyp)^2)
                        if d <= fov then
                            local score = d
                            if (s.aim_target_type or 0) == 1 then
                                local p = e.hrp.Position
                                score = (p.X-cam.X)^2 + (p.Y-cam.Y)^2 + (p.Z-cam.Z)^2
                            end
                            if not bestscore or score < bestscore then
                                bestscore, best = score, e
                            end
                        end
                    end
                end
            end
        end
        chosen = best
        if s.aim_lock then aim_locked = best end
    end
    if not chosen then return end
    local hp = hitbox_pos(chosen, cam)
    if not hp then return end
    local sx, sy, v = draw.WorldToScreen(hp.X, hp.Y, hp.Z)
    if not v then return end
    aim_target = {sx = sx, sy = sy}
    local smooth = s.aim_smooth or 5
    if smooth <= 0 then
        input.MoveMouse(sx - cxp, sy - cyp)
    else
        input.MoveMouse((sx-cxp)/smooth, (sy-cyp)/smooth)
    end
end

local function draw_aim_visuals()
    if not s.aim_enabled then return end
    local cxp, cyp = fov_center()
    if s.aim_fov_show then
        draw.Circle(cxp, cyp, s.aim_fov or 120, s.aim_fov_show_color or {1,1,1,1}, 64, 1)
    end
    if s.aim_line and aim_target then
        styled_line(cxp, cyp, aim_target.sx, aim_target.sy,
            s.aim_line_color or {1,0,0,1}, s.aim_line_style or 0)
    end
    if s.bal_enabled and bal_marker and s.bal_show_marker then
        local col = s.bal_show_marker_color or {1, 0.5, 0, 1}
        local mx, my = bal_marker.sx, bal_marker.sy
        local arm = 6
        draw.Line(mx - arm, my, mx + arm, my, col, 2)
        draw.Line(mx, my - arm, mx, my + arm, col, 2)
    end
    if s.bal_enabled and bal_marker and s.bal_show_text then
        local txt = string.format("%.0fm  drop %.2fm", bal_marker.meters, bal_marker.drop_m)
        local fs = 12
        local tw = draw.GetTextSize(txt, fs)
        draw.Text(bal_marker.sx - tw * 0.5, bal_marker.sy - 18, txt, {1, 0.85, 0.4, 1}, fs)
    end
end

-- ═══════════════════════════════════════════════════════════════════════
-- WEAPON HUD
-- ═══════════════════════════════════════════════════════════════════════

local WH_COLORS = {
    label = {0.7,0.7,0.75,1}, name = {1,1,1,1}, empty = {0.5,0.5,0.5,1},
    bg = {0.06,0.06,0.07,0.85}, dbg_bg = {0.04,0.04,0.05,0.85},
    ok = {0.5,0.9,0.5,1}, fail = {0.9,0.5,0.5,1}, dim = {0.7,0.7,0.75,1}
}

local function draw_weapon_hud()
    if not s.wh_enabled then return end
    local fs = tonumber(s.wh_font) or 16
    local x = tonumber(s.wh_x) or 20
    local y = tonumber(s.wh_y) or 20
    local weapon_name, source = get_local_weapon()
    local display = weapon_name or "None"
    local label = "Weapon:"
    local label_w = draw.GetTextSize(label, fs)
    local value_w = draw.GetTextSize(display, fs)
    local pad_x, pad_y = 10, 6
    local total_w = label_w + value_w + 12
    local total_h = fs + pad_y * 2
    draw.RectFilled(x - pad_x, y - pad_y, total_w + pad_x*2, total_h, WH_COLORS.bg, 4)
    draw.Text(x, y, label, WH_COLORS.label, fs)
    local value_color = weapon_name and WH_COLORS.name or WH_COLORS.empty
    draw.Text(x + label_w + 8, y, display, value_color, fs)
    local next_y = y + total_h + 4
    if s.wh_debug then
        local dbg_fs = fs - 2
        if dbg_fs < 10 then dbg_fs = 10 end
        local status_line
        if weapon_name then
            status_line = "  Status: FOUND -> " .. weapon_name
        else
            status_line = "  Status: no weapon detected"
        end
        local auto_combo = weapon_to_combo(AUTO_INDEX)
        local combo_val = tonumber(s.bal_weapon) or auto_combo
        local is_auto = (combo_val == auto_combo)
        local prof_name
        if is_auto then
            local m = find_weapon_index_by_name(weapon_name)
            prof_name = m and ("Auto -> " .. WEAPONS[m].name) or "Auto (no match)"
        else
            local w_idx = combo_to_weapon(combo_val)
            prof_name = WEAPONS[w_idx] and WEAPONS[w_idx].name or "?"
        end
        local mode_line = bal_auto_mode and "  Mode: Auto" or "  Mode: Manual"
        local lines = {
            status_line,
            "  Source: " .. (source or "--"),
            "  Ballistics: " .. prof_name,
            mode_line,
        }
        local max_w = 0
        for i = 1, #lines do
            local w = draw.GetTextSize(lines[i], dbg_fs)
            if w > max_w then max_w = w end
        end
        local line_h = dbg_fs + 4
        local bg_h = #lines * line_h + 8
        draw.RectFilled(x - pad_x, next_y - 4, max_w + pad_x*2, bg_h, WH_COLORS.dbg_bg, 4)
        for i = 1, #lines do
            local color
            if i == 1 then color = weapon_name and WH_COLORS.ok or WH_COLORS.fail
            elseif i == 3 then color = is_auto and {0.6, 0.8, 1, 1} or WH_COLORS.dim
            elseif i == 4 then color = bal_auto_mode and {0.6, 0.8, 1, 1} or {1, 0.7, 0.4, 1}
            else color = WH_COLORS.dim end
            draw.Text(x, next_y + (i-1)*line_h, lines[i], color, dbg_fs)
        end
    end
end

-- ═══════════════════════════════════════════════════════════════════════
-- DESYNC MARKER
-- ═══════════════════════════════════════════════════════════════════════

local ds_marker_pos = nil
local ds_last_toggle = 0
local ds_last_mode = 0
local ds_toggle_pressed = false
local ds_mode_pressed = false
local ds_camera_mode = 2

local function ds_get_player_position()
    local cam = camera.GetPosition()
    if not cam then return nil end
    local fb = DS_OFFSET_FB
    local lr = DS_OFFSET_LR
    if ds_camera_mode == 1 then
        return {x = cam.X, y = cam.Y, z = cam.Z}
    end
    local look = camera.GetLookVector()
    if not look then return cam end
    local right_x = -look.Z
    local right_y = 0
    local right_z = look.X
    return {
        x = cam.X + look.X * fb + right_x * lr,
        y = cam.Y + look.Y * fb + right_y * lr,
        z = cam.Z + look.Z * fb + right_z * lr,
    }
end

local function draw_desync_marker()
    if not s.ds_enabled then return end
    local now = utility.GetTickCount()
    local toggle_key = tonumber(s.ds_toggle_key) or DS_DEFAULT_TOGGLE
    local mode_key = tonumber(s.ds_mode_key) or DS_DEFAULT_MODE
    local toggle_down = false
    pcall(function() toggle_down = input.IsKeyDown(toggle_key) end)
    local mode_down = false
    pcall(function() mode_down = input.IsKeyDown(mode_key) end)
    if toggle_down and not ds_toggle_pressed and now - ds_last_toggle > 300 then
        ds_toggle_pressed = true
        ds_last_toggle = now
        if ds_marker_pos then
            ds_marker_pos = nil
            if notify then notify.Warning("Desync", "Marker cleared") end
        else
            local pos = ds_get_player_position()
            if pos then
                ds_marker_pos = pos
                if notify then notify.Success("Desync", "Marker saved") end
            end
        end
    elseif not toggle_down then
        ds_toggle_pressed = false
    end
    if mode_down and not ds_mode_pressed and now - ds_last_mode > 300 then
        ds_mode_pressed = true
        ds_last_mode = now
        ds_camera_mode = (ds_camera_mode == 1) and 2 or 1
        if notify then notify.Success("Desync", ds_camera_mode == 1 and "1st Person" or "3rd Person") end
    elseif not mode_down then
        ds_mode_pressed = false
    end
    local player_pos = ds_get_player_position()
    if ds_marker_pos then
        local dist = 0
        if player_pos then
            local dx = ds_marker_pos.x - player_pos.x
            local dy = ds_marker_pos.y - player_pos.y
            local dz = ds_marker_pos.z - player_pos.z
            dist = sqrt(dx*dx + dy*dy + dz*dz)
        end
        local dist_m = dist / STUDS_PER_METER
        local radius_m = DS_CIRCLE_RADIUS
        local radius_studs = radius_m * STUDS_PER_METER
        local outside = dist_m > radius_m
        if s.ds_show_circle then
            local segments = DS_CIRCLE_SEGMENTS
            local circle_col = s.ds_show_circle_color or {0.7, 0.3, 1, 0.6}
            local prev_sx, prev_sy, prev_valid = nil, nil, false
            for i = 0, segments do
                local angle = (i / segments) * math.pi * 2
                local wx = ds_marker_pos.x + math.cos(angle) * radius_studs
                local wz = ds_marker_pos.z + math.sin(angle) * radius_studs
                local wy = ds_marker_pos.y
                local csx, csy, cvalid = draw.WorldToScreen(wx, wy, wz)
                if cvalid and prev_valid then
                    draw.Line(prev_sx, prev_sy, csx, csy, circle_col, 2)
                end
                prev_sx, prev_sy, prev_valid = csx, csy, cvalid
            end
        end
        local sx, sy, on_screen = draw.WorldToScreen(ds_marker_pos.x, ds_marker_pos.y, ds_marker_pos.z)
        if on_screen then
            local mc = s.ds_show_label_color or {1, 0, 1, 1}
            if outside and s.ds_show_outside_warning then mc = {1, 0.2, 0.2, 1} end
            draw.Circle(sx, sy, 12, mc, 32, 2)
            draw.Circle(sx, sy, 6, mc, 16, 2)
            if s.ds_show_label then
                local txt1 = outside and "OUT OF RANGE" or "DESYNC"
                local txt2 = string.format("%.1fm / %dm", dist_m, radius_m)
                local tw1 = draw.GetTextSize(txt1, 14)
                local tw2 = draw.GetTextSize(txt2, 12)
                draw.Text(sx - tw1 * 0.5, sy - 44, txt1, mc, 14)
                draw.Text(sx - tw2 * 0.5, sy - 28, txt2, {mc[1], mc[2], mc[3], 0.7}, 12)
            end
        end
        if s.ds_show_hud then
            local mc = s.ds_show_label_color or {1, 0, 1, 1}
            local hud_col = outside and {1, 0.2, 0.2, 1} or mc
            draw.RectFilled(10, 10, 260, 95, {0.15, 0, 0.15, 0.9}, 4)
            draw.Text(15, 15, "DESYNC MARKER", hud_col, 14)
            draw.Text(15, 35, string.format("Dist: %.1fm / %dm", dist_m, radius_m), {1, 1, 1, 1}, 12)
            draw.Text(15, 52, string.format("F/T: %d | Lat: %d", DS_OFFSET_FB, DS_OFFSET_LR), {0.8, 0.8, 1, 1}, 11)
            draw.Text(15, 68, ds_camera_mode == 1 and "Mode: 1st Person" or "Mode: 3rd Person", {0.8, 0.8, 1, 1}, 10)
            if outside and s.ds_show_outside_warning then
                draw.Text(15, 82, "! OUT OF RANGE !", {1, 0.2, 0.2, 1}, 10)
            else
                draw.Text(15, 82, "In Range", {0.5, 1, 0.5, 1}, 10)
            end
        end
    elseif s.ds_show_hud then
        draw.RectFilled(10, 10, 240, 40, {0.1, 0.1, 0.1, 0.7}, 4)
        draw.Text(15, 15, "DESYNC MARKER", {0.5, 0.5, 0.5, 1}, 14)
        draw.Text(15, 32, "Press Toggle Key", {0.7, 0.7, 0.7, 1}, 11)
    end
end

-- ═══════════════════════════════════════════════════════════════════════
-- RADAR
-- ═══════════════════════════════════════════════════════════════════════

local radar_tx, radar_ty = nil, nil
local radar_dx_draw, radar_dy_draw = nil, nil
local radar_dragging = false
local radar_grab_dx, radar_grab_dy = 0, 0

local function radar_mouse()
    if input and input.GetMousePosition then
        local okm, mx, my = pcall(input.GetMousePosition)
        if okm and mx then return mx, my end
    end
    local oku, ux, uy = pcall(function() return utility.GetMousePos() end)
    if oku and ux then return ux, uy end
    return nil, nil
end

local function draw_radar(cam)
    if not s.radar_enabled then return end
    local look
    local okl, lv = pcall(camera.GetLookVector)
    if okl and lv then look = lv end
    if not look then return end
    local yaw = atan2(look.X, look.Z)
    local sn, cs = sin(yaw), cos(yaw)
    local size = tonumber(s.radar_size) or 300
    local rad = size * 0.5
    local range_m = tonumber(s.radar_range) or 200
    local range = range_m * STUDS_PER_METER
    local range_sq = range * range
    local margin = 14
    if not radar_tx then radar_tx, radar_ty = margin, margin end
    if not radar_dx_draw then radar_dx_draw, radar_dy_draw = radar_tx, radar_ty end
    local mx, my = radar_mouse()
    local held = input.IsKeyDown and input.IsKeyDown(0x01)
    if mx and held then
        if radar_dragging then
            radar_tx = mx - radar_grab_dx
            radar_ty = my - radar_grab_dy
        elseif mx >= radar_dx_draw and mx <= radar_dx_draw + size
            and my >= radar_dy_draw and my <= radar_dy_draw + size then
            radar_dragging = true
            radar_grab_dx = mx - radar_tx
            radar_grab_dy = my - radar_ty
        end
    else
        radar_dragging = false
    end
    radar_dx_draw = radar_dx_draw + (radar_tx - radar_dx_draw) * 0.08
    radar_dy_draw = radar_dy_draw + (radar_ty - radar_dy_draw) * 0.08
    local ox, oy = radar_dx_draw, radar_dy_draw
    local ccx, ccy = ox + rad, oy + rad
    local bg = {0.06,0.06,0.07,1}
    local ring = {0.35,0.35,0.38,1}
    local edge = {0.55,0.55,0.60,1}
    local tick = {0.75,0.75,0.78,1}
    local mecol = {0.85,0.85,0.88,1}
    local lblc = {0.6,0.6,0.63,1}
    draw.CircleFilled(ccx, ccy, rad, bg, 72)
    draw.Circle(ccx, ccy, rad*0.33, ring, 48, 1)
    draw.Circle(ccx, ccy, rad*0.66, ring, 56, 1)
    draw.Circle(ccx, ccy, rad, edge, 72, 2)
    local tl = 8
    draw.Line(ccx, oy, ccx, oy+tl, tick, 2)
    draw.Line(ccx, oy+size-tl, ccx, oy+size, tick, 2)
    draw.Line(ox, ccy, ox+tl, ccy, tick, 2)
    draw.Line(ox+size-tl, ccy, ox+size, ccy, tick, 2)
    local rt = s.radar_targets or {}
    local show_p, show_z
    if rt[0] ~= nil then
        show_p, show_z = rt[0] == true, rt[1] == true
    else
        show_p, show_z = rt[1] == true, rt[2] == true
    end
    local dot = tonumber(s.radar_dot) or 3
    local scale = rad / range
    local clip = rad - dot - 1
    local clip_sq = clip * clip
    local cam_x, cam_y, cam_z = cam.X, cam.Y, cam.Z
    for i = 1, entry_count do
        local e = entries[i]
        local match = (e.kind == "player" and show_p) or (e.kind == "zombie" and show_z)
        if match and utility.IsValid(e.hrp) and not is_local_entity(e) then
            local p = e.hrp.Position
            if p then
                local dx = p.X - cam_x
                local dz = p.Z - cam_z
                if dx*dx + dz*dz <= range_sq then
                    local rx = dx*cs - dz*sn
                    local rz = dx*sn + dz*cs
                    local px = ccx - rx*scale
                    local py = ccy - rz*scale
                    local ddx, ddy = px - ccx, py - ccy
                    if ddx*ddx + ddy*ddy <= clip_sq then
                        local col
                        if e.kind == "player" then col = {0.55,0.75,1,1}
                        elseif e.is_special then col = {1,0.8,0,1}
                        else col = {1,0.35,0.35,1} end
                        draw.CircleFilled(px, py, dot, col, 14)
                    end
                end
            end
        end
    end
    local tip, wid = 6, 4
    local nose = {ccx, ccy - tip}
    local bl = {ccx - wid, ccy + tip*0.8}
    local br = {ccx + wid, ccy + tip*0.8}
    draw.PolyFilled({nose, bl, br}, mecol)
    draw.Line(nose[1], nose[2], bl[1], bl[2], {0.15,0.15,0.15,1}, 1.5)
    draw.Line(nose[1], nose[2], br[1], br[2], {0.15,0.15,0.15,1}, 1.5)
    draw.Line(bl[1], bl[2], br[1], br[2], {0.15,0.15,0.15,1}, 1.5)
    local fs = 10
    local function ring_label(frac)
        local txt = tostring(floor(range_m * frac)) .. "m"
        local tw, th = draw.GetTextSize(txt, fs)
        local ly = ccy + rad * frac - th * 0.5 - 3
        draw.RectFilled(ccx - tw*0.5 - 4, ly - 2, tw + 8, th + 4, bg, 3)
        draw.Text(ccx - tw*0.5, ly, txt, lblc, fs)
    end
    ring_label(0.33)
    ring_label(0.66)
    ring_label(1.0)
end

-- ═══════════════════════════════════════════════════════════════════════
-- CONFIG SAVE/LOAD
-- ═══════════════════════════════════════════════════════════════════════

local CONFIG_NAME = "aftermath_config.txt"

local function get_config_paths()
    local paths = {}
    local appdata = os.getenv("LOCALAPPDATA")
    if appdata then
        paths[#paths + 1] = appdata .. "\\" .. CONFIG_NAME
        paths[#paths + 1] = appdata .. "\\Project Vector\\" .. CONFIG_NAME
    end
    local userprofile = os.getenv("USERPROFILE")
    if userprofile then
        paths[#paths + 1] = userprofile .. "\\" .. CONFIG_NAME
        paths[#paths + 1] = userprofile .. "\\Desktop\\" .. CONFIG_NAME
    end
    paths[#paths + 1] = CONFIG_NAME
    paths[#paths + 1] = ".\\" .. CONFIG_NAME
    return paths
end

local function save_config()
    local content_parts = {}
    content_parts[#content_parts + 1] = "-- Aftermath Config\n\n"
    for _, tab in ipairs(base_ui.state.tabs) do
        for _, group in ipairs(tab.groups) do
            for _, w in ipairs(group.widgets) do
                if w.type == "checkbox" then
                    content_parts[#content_parts + 1] = w.id .. "=" .. (base_ui.state.values[w.id] and "1" or "0") .. "\n"
                    if base_ui.state.colors[w.id] then
                        local c = base_ui.state.colors[w.id]
                        content_parts[#content_parts + 1] = w.id .. "_color=" ..
                            string.format("%.4f,%.4f,%.4f,%.4f", c[1] or 1, c[2] or 1, c[3] or 1, c[4] or 1) .. "\n"
                    end
                elseif w.type == "slider_int" or w.type == "slider_float" then
                    content_parts[#content_parts + 1] = w.id .. "=" .. tostring(tonumber(base_ui.state.values[w.id]) or 0) .. "\n"
                elseif w.type == "combo" then
                    content_parts[#content_parts + 1] = w.id .. "=" .. tostring(tonumber(base_ui.state.values[w.id]) or 0) .. "\n"
                elseif w.type == "hotkey" then
                    content_parts[#content_parts + 1] = w.id .. "_key=" .. tostring(base_ui.state.keys[w.id] or 0) .. "\n"
                elseif w.type == "color" then
                    local c = base_ui.state.colors[w.id] or {1,1,1,1}
                    content_parts[#content_parts + 1] = w.id .. "_color=" ..
                        string.format("%.4f,%.4f,%.4f,%.4f", c[1] or 1, c[2] or 1, c[3] or 1, c[4] or 1) .. "\n"
                elseif w.type == "multi" then
                    local vals = base_ui.state.values[w.id] or {}
                    local packed = {}
                    for i = 1, #w.items do packed[i] = vals[i] and "1" or "0" end
                    content_parts[#content_parts + 1] = w.id .. "=" .. table.concat(packed) .. "\n"
                end
            end
        end
    end
    content_parts[#content_parts + 1] = "\n-- Weapon drop multipliers --\n"
    for i, w in ipairs(WEAPONS) do
        content_parts[#content_parts + 1] = "weapon_" .. i .. "_drop_mult=" .. tostring(w.drop_mult or 1.13) .. "\n"
    end
    local content = table.concat(content_parts)
    local paths = get_config_paths()
    for _, path in ipairs(paths) do
        local ok = pcall(function()
            local file = io.open(path, "w")
            if not file then return false end
            file:write(content)
            file:close()
            return true
        end)
        if ok then
            print("[Aftermath] Config saved to: " .. path)
            if notify then notify.Success("Aftermath", "Config saved") end
            return true
        end
    end
    if notify then notify.Warning("Aftermath", "Failed to save config") end
    return false
end

local function load_config()
    local paths = get_config_paths()
    local content = nil
    for _, path in ipairs(paths) do
        local ok, c = pcall(function()
            local file = io.open(path, "r")
            if not file then return nil end
            local data = file:read("*a")
            file:close()
            return data
        end)
        if ok and c and #c > 0 then
            content = c
            break
        end
    end
    if not content then return false end
    local data = {}
    for line in content:gmatch("[^\r\n]+") do
        local key, value = line:match("^([^=]+)=(.*)$")
        if key and value then data[key] = value end
    end
    for _, tab in ipairs(base_ui.state.tabs) do
        for _, group in ipairs(tab.groups) do
            for _, w in ipairs(group.widgets) do
                if data[w.id] then
                    if w.type == "checkbox" then
                        base_ui.state.values[w.id] = data[w.id] == "1"
                    elseif w.type == "slider_int" or w.type == "slider_float" then
                        base_ui.state.values[w.id] = tonumber(data[w.id]) or 0
                    elseif w.type == "combo" then
                        base_ui.state.values[w.id] = tonumber(data[w.id]) or 0
                    elseif w.type == "multi" then
                        local packed = data[w.id]
                        local out = {}
                        for i = 1, #packed do out[i] = packed:sub(i, i) == "1" end
                        base_ui.state.values[w.id] = out
                    end
                end
                local ckey = w.id .. "_color"
                if data[ckey] then
                    local r, g, b, a = data[ckey]:match("([%d%.%-]+),([%d%.%-]+),([%d%.%-]+),([%d%.%-]+)")
                    if r then
                        base_ui.state.colors[w.id] = {tonumber(r), tonumber(g), tonumber(b), tonumber(a)}
                    end
                end
                local kkey = w.id .. "_key"
                if data[kkey] then
                    base_ui.state.keys[w.id] = tonumber(data[kkey]) or 0
                end
            end
        end
    end
    for i, w in ipairs(WEAPONS) do
        local key = "weapon_" .. i .. "_drop_mult"
        if data[key] then
            w.drop_mult = tonumber(data[key]) or w.drop_mult
        end
    end
    return true
end

base_ui:add_button("Config", "Config", "cfg_save", "Save Config", save_config)
base_ui:add_button("Config", "Config", "cfg_load", "Load Config", load_config)

-- ═══════════════════════════════════════════════════════════════════════
-- ONFRAME
-- ═══════════════════════════════════════════════════════════════════════

OnFrame = function()
    sw, sh = draw.GetScreenSize()

    sync_settings()
    autoswitch_ballistics()
    update_visibility()

    if base_ui.state.keys["ui_toggle_key"] then
        base_ui.state.toggle_key = base_ui.state.keys["ui_toggle_key"]
    end

    process_hotkey_listening(base_ui)
    process_toggle(base_ui)

    scan_vehicles()

    local okc, cam = pcall(camera.GetPosition)
    if not okc or not cam then
        draw_window(base_ui)
        return
    end

    pcall(update_local_addr, cam)

    if entry_count > 0 then
        pcall(do_aimbot, cam)
        draw_aim_visuals()
    end

    pcall(draw_radar, cam)
    pcall(draw_weapon_hud)
    pcall(draw_vehicles, cam)
    pcall(draw_desync_marker)

    local t = s.targets or {}
    local show_players = t[1] == true
    local show_zombies = t[2] == true
    if s.esp_enabled and entry_count > 0 and (show_players or show_zombies) then
        local only_special = s.only_special == true
        local max_sq = (tonumber(s.max_distance) or 5000) ^ 2
        local cx, cy, cz = cam.X, cam.Y, cam.Z
        for i = 1, entry_count do
            local e = entries[i]
            local hrp = e.hrp
            if hrp and hrp.Address then
                local match = (e.kind == "player" and show_players) or (e.kind == "zombie" and show_zombies)
                if only_special and e.kind == "zombie" and not e.is_special then
                    match = false
                end
                if match and not is_local_entity(e) then
                    local hp = hrp.Position
                    if hp then
                        local dx, dy, dz = hp.X-cx, hp.Y-cy, hp.Z-cz
                        local dsq = dx*dx + dy*dy + dz*dz
                        if dsq <= max_sq then
                            render_entity(e, cam)
                        end
                    end
                end
            end
        end
    end

    draw_window(base_ui)
end

base_ui:apply_ui_settings()
pcall(load_config)
base_ui:apply_ui_settings()

if notify and notify.Success then
    notify.Success("Aftermath v" .. VERSION, "Loaded Successfully")
else
    print("Aftermath Loaded Successfully")
end
