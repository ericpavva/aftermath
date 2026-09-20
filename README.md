local VERSION = "1.992"

if not draw or not menu or not utility or not camera or not input or not thread then
    if notify and notify.error then
        notify.error("Aftermath", "Incompatible Vector build")
    else
        print("[Aftermath] missing required API")
    end
    return
end

local ok, err =
    pcall(
    function()
        local sqrt, floor, min, max = math.sqrt, math.floor, math.min, math.max
        local abs = math.abs
        local sin, cos, atan2 = math.sin, math.cos, (math.atan2 or math.atan)

        local BOX_TYPE = {STANDARD = 0, CORNER = 1, THREE_D = 2}
        local SNAP_MODE = {TOP = 0, CENTER = 1, BOTTOM = 2}
        local LINE_STYLE = {SOLID = 0, DASHED = 1, FADE = 2}

        local STUDS_PER_METER = 2.6

        local GAME_GRAVITY_STUDS = 85
        pcall(function()
            if game.Workspace and game.Workspace.Gravity then
                GAME_GRAVITY_STUDS = game.Workspace.Gravity
            end
        end)
        local GAME_GRAVITY_MPS = GAME_GRAVITY_STUDS / STUDS_PER_METER

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
        }

        local WEAPON_COMBO = {}
        for i, w in ipairs(WEAPONS) do
            WEAPON_COMBO[i] = w.name
        end
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
                        if string.find(lower_name, pat, 1, true) then
                            return i
                        end
                    end
                end
            end

            for i, w in ipairs(WEAPONS) do
                if w.name and string.lower(w.name) == lower_name then
                    return i
                end
            end

            return nil
        end

        menu.add_tab("Aftermath", "AM", "full")
        menu.add_group("Aftermath", "Entity Visuals")
        menu.add_group("Aftermath", "Entity Aimbot", 0, true)
        menu.add_group("Aftermath", "Ballistics")
        menu.add_group("Aftermath", "Radar")
        menu.add_group("Aftermath", "Weapon HUD")

        local menu_items = {
            {t = "checkbox", id = "esp_enabled", n = "Enable ESP", v = false},
            {t = "multicombo", id = "targets", n = "Targets", o = {"Players", "Zombies"}, dv = {true, false}},
            {t = "checkbox", id = "disable_zombie_scan", n = "Disable Zombie Scan", v = false},
            {t = "checkbox", id = "box", n = "Box", v = false, p = "esp_enabled", c = {1, 1, 1, 1}},
            {t = "combo", id = "box_type", n = "Box Style", o = {"2D", "Corner", "3D"}, v = 0, p = "box"},
            {t = "checkbox", id = "fill", n = "Box Filled", v = false, p = "esp_enabled", c = {1, 1, 1, 0.2}},
            {t = "slider_int", id = "fill_op", n = "Fill Opacity", min = 0, max = 100, v = 20, p = "fill"},
            {t = "checkbox", id = "name", n = "Name", v = false, p = "esp_enabled", c = {1, 1, 1, 1}},
            {t = "checkbox", id = "distance", n = "Distance", v = false, p = "esp_enabled", c = {0.7, 0.7, 0.7, 1}},
            {t = "checkbox", id = "skeleton", n = "Skeleton", v = false, p = "esp_enabled", c = {1, 0.8, 0.2, 1}},
            {t = "checkbox", id = "head_dot", n = "Head Dot", v = false, p = "esp_enabled", c = {1, 1, 1, 1}},
            {t = "checkbox", id = "viewline", n = "View Line", v = false, p = "esp_enabled", c = {1, 1, 1, 1}},
            {t = "slider_int", id = "vl_length", n = "View Line Length", min = 1, max = 15, v = 5, p = "viewline"},
            {t = "combo", id = "vl_style", n = "View Line Style", o = {"Solid", "Dashed", "Fade"}, v = 2, p = "viewline"},
            {t = "checkbox", id = "snapline", n = "Snaplines", v = false, p = "esp_enabled", c = {1, 1, 1, 0.5}},
            {t = "combo", id = "snap_mode", n = "Snapline Origin", o = {"Top", "Center", "Bottom"}, v = 2, p = "snapline"},
            {t = "slider_int", id = "max_distance", n = "Render Distance", min = 1, max = 5000, v = 5000, p = "esp_enabled"},
            {t = "slider_int", id = "font_size", n = "Text Size", min = 8, max = 24, v = 13, p = "esp_enabled"},

            {t = "checkbox", id = "veh_enabled", n = "Enable Vehicle ESP", v = false},
            {t = "checkbox", id = "veh_name", n = "Vehicle Name", v = true, p = "veh_enabled", c = {1, 0.8, 0.2, 1}},
            {t = "checkbox", id = "veh_dist", n = "Vehicle Distance", v = true, p = "veh_enabled", c = {0.7, 0.7, 0.7, 1}},
            {t = "checkbox", id = "veh_box", n = "Vehicle Box", v = false, p = "veh_enabled", c = {1, 0.5, 0, 1}},
            {t = "slider_int", id = "veh_range", n = "Vehicle Range (m)", min = 10, max = 2000, v = 500, p = "veh_enabled"},
            {t = "slider_int", id = "veh_font", n = "Vehicle Font Size", min = 8, max = 24, v = 14, p = "veh_enabled"},

            {g = "Entity Aimbot", t = "checkbox", id = "aim_enabled", n = "Enable Aimbot", v = false},
            {g = "Entity Aimbot", t = "checkbox", id = "aim_zombies", n = "Zombie Aimbot", v = false, p = "aim_enabled"},
            {g = "Entity Aimbot", t = "hotkey", id = "aim_key", n = "Aimbot Keybind", k = 0x02, p = "aim_enabled"},
            {g = "Entity Aimbot", t = "combo", id = "aim_target_type", n = "Target Priority", o = {"Crosshair", "Distance"}, v = 0, p = "aim_enabled"},
            {g = "Entity Aimbot", t = "slider_int", id = "aim_smooth", n = "Smoothing (0 = Instant)", min = 0, max = 20, v = 5, p = "aim_enabled"},
            {g = "Entity Aimbot", t = "combo", id = "aim_hitbox", n = "Hitbox", o = {"Head", "Torso", "Left Arm", "Right Arm", "Left Leg", "Right Leg", "Closest"}, v = 0, p = "aim_enabled"},
            {g = "Entity Aimbot", t = "checkbox", id = "aim_predict", n = "Prediction (XYZ)", v = false, p = "aim_enabled"},
            {g = "Entity Aimbot", t = "checkbox", id = "aim_fov_show", n = "Show FOV Circle", v = true, p = "aim_enabled", c = {1, 1, 1, 1}},
            {g = "Entity Aimbot", t = "slider_int", id = "aim_fov", n = "FOV", min = 1, max = 500, v = 120, p = "aim_enabled"},
            {g = "Entity Aimbot", t = "slider_int", id = "aim_max_dist", n = "Max Distance (Player)", min = 1, max = 5000, v = 1000, p = "aim_enabled"},
            {g = "Entity Aimbot", t = "slider_int", id = "aim_zombie_dist", n = "Max Distance (Zombie)", min = 1, max = 5000, v = 1000, p = "aim_zombies"},
            {g = "Entity Aimbot", t = "checkbox", id = "aim_lock", n = "Target Lock", v = false, p = "aim_enabled"},
            {g = "Entity Aimbot", t = "checkbox", id = "aim_line", n = "Target Line", v = false, p = "aim_enabled", c = {1, 0, 0, 1}},
            {g = "Entity Aimbot", t = "combo", id = "aim_line_style", n = "Target Line Style", o = {"Solid", "Dashed", "Dotted"}, v = 0, p = "aim_line"},

            {g = "Ballistics", t = "checkbox", id = "bal_enabled", n = "Enable Ballistic Compensation", v = false},
            {g = "Ballistics", t = "combo", id = "bal_weapon", n = "Weapon Profile",
                o = WEAPON_COMBO, v = weapon_to_combo(AUTO_INDEX), p = "bal_enabled"},
            {g = "Ballistics", t = "slider_int", id = "bal_gravity_x100", n = "Gravity (m/s2 x100)", min = 100, max = 5000,
                v = floor(GAME_GRAVITY_MPS * 100), p = "bal_enabled"},
            {g = "Ballistics", t = "slider_int", id = "bal_zero_m", n = "Zero Range (m)", min = 0, max = 400, v = 0, p = "bal_enabled"},
            {g = "Ballistics", t = "slider_int", id = "bal_drop_mult_x100", n = "Drop Multiplier x100", min = 100, max = 200,
                v = 113, p = "bal_enabled"},
            {g = "Ballistics", t = "checkbox", id = "bal_show_marker", n = "Holdover Marker", v = true, p = "bal_enabled", c = {1, 0.5, 0, 1}},
            {g = "Ballistics", t = "checkbox", id = "bal_show_text", n = "Distance / Drop Text", v = true, p = "bal_enabled"},

            {g = "Radar", t = "checkbox", id = "radar_enabled", n = "Enable Radar", v = false},
            {g = "Radar", t = "slider_int", id = "radar_size", n = "Radar Size", min = 80, max = 400, v = 300, p = "radar_enabled"},
            {g = "Radar", t = "multicombo", id = "radar_targets", n = "Radar Targets", o = {"Players", "Zombies"}, dv = {true, true}},
            {g = "Radar", t = "slider_int", id = "radar_range", n = "Range (m)", min = 50, max = 2000, v = 200, p = "radar_enabled"},
            {g = "Radar", t = "slider_int", id = "radar_dot", n = "Dot Size", min = 1, max = 8, v = 3, p = "radar_enabled"},

            {g = "Weapon HUD", t = "checkbox", id = "wh_enabled", n = "Enable Weapon HUD", v = false},
            {g = "Weapon HUD", t = "slider_int", id = "wh_x", n = "HUD X", min = 0, max = 2000, v = 20, p = "wh_enabled"},
            {g = "Weapon HUD", t = "slider_int", id = "wh_y", n = "HUD Y", min = 0, max = 2000, v = 20, p = "wh_enabled"},
            {g = "Weapon HUD", t = "slider_int", id = "wh_font", n = "Font Size", min = 8, max = 32, v = 16, p = "wh_enabled"},
            {g = "Weapon HUD", t = "checkbox", id = "wh_debug", n = "Debug Overlay", v = false, p = "wh_enabled"}
        }

        for _, m in ipairs(menu_items) do
            local grp = m.g or "Entity Visuals"
            local opts = {}
            if m.p then opts.parent = m.p end
            if m.c then opts.colorpicker = m.c end
            if m.t == "checkbox" then
                menu.add_checkbox("Aftermath", grp, m.id, m.n, m.v, opts)
            elseif m.t == "combo" then
                menu.add_combo("Aftermath", grp, m.id, m.n, m.o, m.v, opts)
            elseif m.t == "multicombo" then
                menu.add_multicombo("Aftermath", grp, m.id, m.n, m.o, m.dv)
            elseif m.t == "slider_int" then
                menu.add_slider_int("Aftermath", grp, m.id, m.n, m.min, m.max, m.v, opts)
            elseif m.t == "colorpicker" then
                menu.add_colorpicker("Aftermath", grp, m.id, m.n, m.v, opts)
            elseif m.t == "hotkey" then
                menu.add_hotkey("Aftermath", grp, m.id, m.n, m.k, opts)
            end
        end

        local s = {}
        local function sync_settings()
            for _, m in ipairs(menu_items) do
                if m.t == "colorpicker" then
                    s[m.id] = menu.get_color(m.id)
                elseif m.t == "hotkey" then
                    s[m.id] = menu.get_key(m.id)
                else
                    s[m.id] = menu.get(m.id)
                    if m.c then
                        s[m.id .. "_color"] = menu.get_color(m.id)
                    end
                end
            end
        end

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
                if matched then
                    weapon = WEAPONS[matched]
                end
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
                    menu.set("bal_drop_mult_x100", floor((w.drop_mult or 1.13) * 100))
                end
                return
            end

            local weapon_name = get_local_weapon()
            local matched = find_weapon_index_by_name(weapon_name)

            if matched and matched ~= bal_last_effective then
                bal_last_effective = matched
                local combo = weapon_to_combo(matched)
                bal_last_written = combo
                menu.set("bal_weapon", combo)

                local w = WEAPONS[matched]
                if w then
                    menu.set("bal_drop_mult_x100", floor((w.drop_mult or 1.13) * 100))
                end
            end
        end

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
                        if string.find(lname, "wheel", 1, true) then
                            has_wheel = true
                        end
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
            local now = utility.get_tick_count()
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
                        local sx, sy, on_screen = draw.world_to_screen(pos.X, pos.Y, pos.Z)
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
                                            local world_pos = cf * Vector3.new(
                                                sz.X * 0.5 * ox,
                                                sz.Y * 0.5 * oy,
                                                sz.Z * 0.5 * oz
                                            )
                                            local px, py, pv = draw.world_to_screen(world_pos.X, world_pos.Y, world_pos.Z)
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
                                    draw.rect(mnx, mny, mxx - mnx, mxy - mny, box_col, 0, 1.5)
                                end
                            end

                            local label_parts = {}
                            if show_name then label_parts[#label_parts + 1] = v.name end
                            if show_dist then label_parts[#label_parts + 1] = dist_m .. "m" end

                            if #label_parts > 0 then
                                local txt = table.concat(label_parts, "  ")
                                local tw = draw.get_text_size(txt, fs)
                                local col = show_name and name_col or dist_col
                                draw.text(sx - tw * 0.5, sy - 12, txt, col, fs)
                            end
                        end
                    end
                end
            end
        end

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
                    prev_t = (utility.get_time and utility.get_time()) or os.clock(),
                    samples = {}
                }
                return
            end

            local now = (utility.get_time and utility.get_time()) or os.clock()
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
                if not live[addr] then
                    VELOCITY[addr] = nil
                end
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
                    if h and h.Address == local_addr and utility.is_valid(h) then
                        return
                    end
                end
                local_addr = nil
            end
            local cx, cy, cz = cam.X, cam.Y, cam.Z
            local best, bd = nil, LOCAL_ACQUIRE_SQ
            for i = 1, entry_count do
                local h = entries[i].hrp
                if h and utility.is_valid(h) then
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
            if model:FindFirstChild("UpperTorso") then
                kind, rig = "player", "R15"
            elseif model:FindFirstChild("Torso") then
                if s.disable_zombie_scan then return nil end
                kind, rig = "zombie", "R6"
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
                model = model, body = body, skel = skel
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
                    draw.line(x1 + (x2-x1)*t1, y1 + (y2-y1)*t1,
                              x1 + (x2-x1)*t2, y1 + (y2-y1)*t2, col, 2)
                end
            elseif style == LINE_STYLE.FADE then
                for i = 0, 11 do
                    local t1, t2 = i / 12, (i + 1) / 12
                    local a = (col[4] or 1) * (1 - t1)
                    draw.line(x1 + (x2-x1)*t1, y1 + (y2-y1)*t1,
                              x1 + (x2-x1)*t2, y1 + (y2-y1)*t2,
                              {col[1], col[2], col[3], a}, 2)
                end
            else
                draw.line(x1, y1, x2, y2, col, 2)
            end
        end

        local HAVE_NATIVE_CORNER = type(draw.corner_box) == "function"

        local function corner_box(x, y, w, h, col)
            if HAVE_NATIVE_CORNER then
                draw.corner_box(x, y, w, h, col)
                return
            end
            local cl = min(w, h) * 0.25
            draw.line(x, y, x+cl, y, col, 2)
            draw.line(x, y, x, y+cl, col, 2)
            draw.line(x+w-cl, y, x+w, y, col, 2)
            draw.line(x+w, y, x+w, y+cl, col, 2)
            draw.line(x, y+h-cl, x, y+h, col, 2)
            draw.line(x, y+h, x+cl, y+h, col, 2)
            draw.line(x+w-cl, y+h, x+w, y+h, col, 2)
            draw.line(x+w, y+h-cl, x+w, y+h, col, 2)
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
                local x, y, v = draw.world_to_screen(c[i][1], c[i][2], c[i][3])
                sp[i] = {x, y, v}
            end
            local edges = {
                {1,2},{2,3},{3,4},{4,1},{5,6},{6,7},{7,8},{8,5},
                {1,5},{2,6},{3,7},{4,8}
            }
            for _, e in ipairs(edges) do
                local a, b = sp[e[1]], sp[e[2]]
                if a[3] and b[3] then
                    draw.line(a[1], a[2], b[1], b[2], col, 1.5)
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
                    local px, py, pv = draw.world_to_screen(pos.X, pos.Y, pos.Z)
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

            local col = s.box_color or {1, 1, 1, 1}
            local cx = b.x + b.w * 0.5
            local btype = s.box_type or 0

            local too_small = b.h < 12

            if s.box and b.h > 6 then
                if s.fill then
                    local fc = s.fill_color or {0, 0, 0, 0.4}
                    draw.rect_filled(b.x, b.y, b.w, b.h,
                        {fc[1], fc[2], fc[3], (s.fill_op or 20) * 0.01})
                end
                if btype == BOX_TYPE.CORNER then
                    corner_box(b.x, b.y, b.w, b.h, col)
                elseif btype == BOX_TYPE.THREE_D then
                    draw_3d_box(bb, col)
                else
                    draw.rect(b.x, b.y, b.w, b.h, col, 0, 1.5)
                end
            end

            if s.name then
                local label = e.kind == "player" and "Player" or "Zombie"
                local nc = e.kind == "player" and (s.name_color or {1, 1, 1, 1}) or {1, 0.4, 0.4, 1}
                local fs = s.font_size or 13
                local tw = draw.get_text_size(label, fs)
                draw.text(cx - tw * 0.5, b.y - fs - 2, label, nc, fs)
            end

            if s.skeleton and not too_small then
                local sc = s.skeleton_color or col
                for k = 1, #e.skel do
                    local ap, bp = e.skel[k][1].Position, e.skel[k][2].Position
                    if ap and bp then
                        local ax, ay, av = draw.world_to_screen(ap.X, ap.Y, ap.Z)
                        local bx, by, bv = draw.world_to_screen(bp.X, bp.Y, bp.Z)
                        if av and bv then draw.line(ax, ay, bx, by, sc, 2) end
                    end
                end
            end

            if s.head_dot and not too_small and e.head and e.head.Position then
                local hpos = e.head.Position
                local hx, hy, hv = draw.world_to_screen(hpos.X, hpos.Y, hpos.Z)
                if hv then draw.circle(hx, hy, 4, s.head_dot_color or col, 16, 1.5) end
            end

            if s.viewline then
                local lv = e.hrp.LookVector or e.hrp.look_vector
                local origin = (e.head and e.head.Position) or e.hrp.Position
                if lv and origin then
                    local len = s.vl_length or 5
                    local ex, ey, ez = origin.X + lv.X*len, origin.Y + lv.Y*len, origin.Z + lv.Z*len
                    local sx, sy, v1 = draw.world_to_screen(origin.X, origin.Y, origin.Z)
                    local ex2, ey2, v2 = draw.world_to_screen(ex, ey, ez)
                    if v1 and v2 then
                        styled_line(sx, sy, ex2, ey2, s.viewline_color or col, s.vl_style or 0)
                    end
                end
            end

            if s.snapline then
                local mode = s.snap_mode or SNAP_MODE.BOTTOM
                local snap_y = mode == SNAP_MODE.TOP and 0 or mode == SNAP_MODE.CENTER and sh * 0.5 or sh
                draw.line(sw * 0.5, snap_y, cx, b.y + b.h, s.snapline_color or col, 1.5)
            end

            if s.distance then
                local fs = s.font_size or 13
                local meters = dist / STUDS_PER_METER
                local txt = floor(meters) .. "m"
                local tw = draw.get_text_size(txt, fs)
                draw.text(cx - tw * 0.5, b.y + b.h + 2, txt, s.distance_color or {0.7, 0.7, 0.7, 1}, fs)
            end
        end

        local function fov_center()
            local mx, my
            if utility.get_mouse_pos then
                mx, my = utility.get_mouse_pos()
            elseif input.get_mouse_position then
                mx, my = input.get_mouse_position()
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
            local cam_pos = camera.get_position()
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
            local cam_pos = camera.get_position()
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

            local bx, by, bv = draw.world_to_screen(base.X, base.Y + y_world, base.Z)
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
                        local px, py, v = draw.world_to_screen(p.X, p.Y, p.Z)
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
            if key <= 0 or not input.is_key_down(key) then
                aim_locked = nil
                return
            end

            local cxp, cyp = fov_center()
            local fov = s.aim_fov or 120

            local chosen
            if s.aim_lock and aim_locked and utility.is_valid(aim_locked.hrp)
                and aim_candidate(aim_locked, cam) then
                chosen = aim_locked
            else
                local best, bestscore
                for i = 1, entry_count do
                    local e = entries[i]
                    if utility.is_valid(e.hrp) and aim_candidate(e, cam) then
                        local hp = hitbox_pos(e, cam)
                        if hp then
                            local sx, sy, v = draw.world_to_screen(hp.X, hp.Y, hp.Z)
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
            local sx, sy, v = draw.world_to_screen(hp.X, hp.Y, hp.Z)
            if not v then return end

            aim_target = {sx = sx, sy = sy}

            local smooth = s.aim_smooth or 5
            if smooth <= 0 then
                input.move_mouse(sx - cxp, sy - cyp)
            else
                input.move_mouse((sx-cxp)/smooth, (sy-cyp)/smooth)
            end
        end

        local function draw_aim_visuals()
            if not s.aim_enabled then return end
            local cxp, cyp = fov_center()
            if s.aim_fov_show then
                draw.circle(cxp, cyp, s.aim_fov or 120, s.aim_fov_show_color or {1,1,1,1}, 64, 1)
            end
            if s.aim_line and aim_target then
                styled_line(cxp, cyp, aim_target.sx, aim_target.sy,
                    s.aim_line_color or {1,0,0,1}, s.aim_line_style or 0)
            end

            if s.bal_enabled and bal_marker and s.bal_show_marker then
                local col = s.bal_show_marker_color or {1, 0.5, 0, 1}
                local mx, my = bal_marker.sx, bal_marker.sy
                local arm = 6
                draw.line(mx - arm, my, mx + arm, my, col, 2)
                draw.line(mx, my - arm, mx, my + arm, col, 2)
            end

            if s.bal_enabled and bal_marker and s.bal_show_text then
                local txt = string.format("%.0fm  drop %.2fm", bal_marker.meters, bal_marker.drop_m)
                local fs = 12
                local tw = draw.get_text_size(txt, fs)
                draw.text(bal_marker.sx - tw * 0.5, bal_marker.sy - 18, txt, {1, 0.85, 0.4, 1}, fs)
            end
        end

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
            local label_w = draw.get_text_size(label, fs)
            local value_w = draw.get_text_size(display, fs)

            local pad_x, pad_y = 10, 6
            local total_w = label_w + value_w + 12
            local total_h = fs + pad_y * 2

            draw.rect_filled(x - pad_x, y - pad_y, total_w + pad_x*2, total_h, WH_COLORS.bg, 4)
            draw.text(x, y, label, WH_COLORS.label, fs)
            local value_color = weapon_name and WH_COLORS.name or WH_COLORS.empty
            draw.text(x + label_w + 8, y, display, value_color, fs)

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
                    local w = draw.get_text_size(lines[i], dbg_fs)
                    if w > max_w then max_w = w end
                end
                local line_h = dbg_fs + 4
                local bg_h = #lines * line_h + 8

                draw.rect_filled(x - pad_x, next_y - 4, max_w + pad_x*2, bg_h, WH_COLORS.dbg_bg, 4)

                for i = 1, #lines do
                    local color
                    if i == 1 then
                        color = weapon_name and WH_COLORS.ok or WH_COLORS.fail
                    elseif i == 3 then
                        color = is_auto and {0.6, 0.8, 1, 1} or WH_COLORS.dim
                    elseif i == 4 then
                        color = bal_auto_mode and {0.6, 0.8, 1, 1} or {1, 0.7, 0.4, 1}
                    else
                        color = WH_COLORS.dim
                    end
                    draw.text(x, next_y + (i-1)*line_h, lines[i], color, dbg_fs)
                end
            end
        end

        local function update_visibility()
            if not menu.set_visible then return end
            local master = s.esp_enabled == true
            local dzs = s.disable_zombie_scan == true

            menu.set_visible("targets", master)
            menu.set_visible("box_type", master and s.box == true)
            menu.set_visible("fill_op", master and s.fill == true)
            menu.set_visible("vl_length", master and s.viewline == true)
            menu.set_visible("vl_style", master and s.viewline == true)
            menu.set_visible("snap_mode", master and s.snapline == true)

            local veh = s.veh_enabled == true
            menu.set_visible("veh_name", veh)
            menu.set_visible("veh_dist", veh)
            menu.set_visible("veh_box", veh)
            menu.set_visible("veh_range", veh)
            menu.set_visible("veh_font", veh)

            local am = s.aim_enabled == true
            menu.set_visible("aim_key", am)
            menu.set_visible("aim_zombies", am and not dzs)
            menu.set_visible("aim_target_type", am)
            menu.set_visible("aim_smooth", am)
            menu.set_visible("aim_hitbox", am)
            menu.set_visible("aim_fov_show", am)
            menu.set_visible("aim_fov", am)
            menu.set_visible("aim_max_dist", am)
            menu.set_visible("aim_zombie_dist", am and s.aim_zombies == true and not dzs)
            menu.set_visible("aim_lock", am)
            menu.set_visible("aim_line", am)
            menu.set_visible("aim_line_style", am and s.aim_line == true)
            menu.set_visible("aim_predict", am)

            local bal = s.bal_enabled == true
            menu.set_visible("bal_weapon", bal)
            menu.set_visible("bal_gravity_x100", bal)
            menu.set_visible("bal_zero_m", bal)
            menu.set_visible("bal_drop_mult_x100", bal)
            menu.set_visible("bal_show_marker", bal)
            menu.set_visible("bal_show_text", bal)

            local rm = s.radar_enabled == true
            menu.set_visible("radar_size", rm)
            menu.set_visible("radar_targets", rm)
            menu.set_visible("radar_range", rm)
            menu.set_visible("radar_dot", rm)

            local wh = s.wh_enabled == true
            menu.set_visible("wh_x", wh)
            menu.set_visible("wh_y", wh)
            menu.set_visible("wh_font", wh)
            menu.set_visible("wh_debug", wh)
        end

        local radar_tx, radar_ty = nil, nil
        local radar_dx_draw, radar_dy_draw = nil, nil
        local radar_dragging = false
        local radar_grab_dx, radar_grab_dy = 0, 0

        local function radar_mouse()
            if input and input.get_mouse_position then
                local okm, mx, my = pcall(input.get_mouse_position)
                if okm and mx then return mx, my end
            end
            local oku, ux, uy = pcall(function() return utility.get_mouse_pos() end)
            if oku and ux then return ux, uy end
            return nil, nil
        end

        local function draw_radar(cam)
            if not s.radar_enabled then return end
            local look
            local okl, lv = pcall(camera.get_look_vector)
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
            local held = input.is_key_down and input.is_key_down(0x01)
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

            draw.circle_filled(ccx, ccy, rad, bg, 72)
            draw.circle(ccx, ccy, rad*0.33, ring, 48, 1)
            draw.circle(ccx, ccy, rad*0.66, ring, 56, 1)
            draw.circle(ccx, ccy, rad, edge, 72, 2)

            local tl = 8
            draw.line(ccx, oy, ccx, oy+tl, tick, 2)
            draw.line(ccx, oy+size-tl, ccx, oy+size, tick, 2)
            draw.line(ox, ccy, ox+tl, ccy, tick, 2)
            draw.line(ox+size-tl, ccy, ox+size, ccy, tick, 2)

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
                if match and utility.is_valid(e.hrp) and not is_local_entity(e) then
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
                                local col = e.kind == "player" and {0.55,0.75,1,1} or {1,0.35,0.35,1}
                                draw.circle_filled(px, py, dot, col, 14)
                            end
                        end
                    end
                end
            end

            local tip, wid = 6, 4
            local nose = {ccx, ccy - tip}
            local bl = {ccx - wid, ccy + tip*0.8}
            local br = {ccx + wid, ccy + tip*0.8}
            draw.poly_filled({nose, bl, br}, mecol)
            draw.line(nose[1], nose[2], bl[1], bl[2], {0.15,0.15,0.15,1}, 1.5)
            draw.line(nose[1], nose[2], br[1], br[2], {0.15,0.15,0.15,1}, 1.5)
            draw.line(bl[1], bl[2], br[1], br[2], {0.15,0.15,0.15,1}, 1.5)

            local fs = 10
            local function ring_label(frac)
                local txt = tostring(floor(range_m * frac)) .. "m"
                local tw, th = draw.get_text_size(txt, fs)
                local ly = ccy + rad * frac - th * 0.5 - 3
                draw.rect_filled(ccx - tw*0.5 - 4, ly - 2, tw + 8, th + 4, bg, 3)
                draw.text(ccx - tw*0.5, ly, txt, lblc, fs)
            end
            ring_label(0.33)
            ring_label(0.66)
            ring_label(1.0)
        end

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

        local SAVE_ITEMS = {}
        for _, m in ipairs(menu_items) do
            local entry = { id = m.id }
            if m.t == "checkbox" then
                entry.type = "bool"
                if m.c then entry.color = true end
            elseif m.t == "slider_int" then
                entry.type = "int"
            elseif m.t == "combo" then
                entry.type = "int"
            elseif m.t == "colorpicker" then
                entry.type = "color"
            elseif m.t == "hotkey" then
                entry.type = "key"
            elseif m.t == "multicombo" then
                entry.type = "multibool"
            end
            if entry.type then
                SAVE_ITEMS[#SAVE_ITEMS + 1] = entry
            end
        end

        local function save_config()
            local config = {}
            config.version = VERSION

            for _, item in ipairs(SAVE_ITEMS) do
                local id = item.id
                if item.type == "bool" then
                    config[id] = s[id] and "1" or "0"
                    if item.color then
                        local col = s[id .. "_color"] or {1, 1, 1, 1}
                        config[id .. "_color"] = string.format("%.4f,%.4f,%.4f,%.4f",
                            col[1] or 1, col[2] or 1, col[3] or 1, col[4] or 1)
                    end
                elseif item.type == "int" then
                    config[id] = tostring(tonumber(s[id]) or 0)
                elseif item.type == "color" then
                    local col = menu.get_color(id)
                    config[id .. "_color"] = string.format("%.4f,%.4f,%.4f,%.4f",
                        col[1] or 1, col[2] or 1, col[3] or 1, col[4] or 1)
                elseif item.type == "key" then
                    config[id .. "_key"] = tostring(tonumber(menu.get_key(id)) or 0)
                elseif item.type == "multibool" then
                    local v = s[id] or {}
                    local packed = {}
                    for i = 1, 20 do
                        packed[i] = v[i] and "1" or "0"
                    end
                    config[id] = table.concat(packed)
                end
            end

            config.weapons = {}
            for i, w in ipairs(WEAPONS) do
                config.weapons[i] = tostring(w.drop_mult or 1.13)
            end

            local content_parts = {}
            content_parts[#content_parts + 1] = "-- Aftermath Config\n"
            content_parts[#content_parts + 1] = "-- Generated automatically\n\n"

            for id, value in pairs(config) do
                if id ~= "weapons" and id ~= "version" then
                    content_parts[#content_parts + 1] = id .. "=" .. value .. "\n"
                end
            end

            content_parts[#content_parts + 1] = "\n-- Weapon drop multipliers --\n"
            for i, val in ipairs(config.weapons) do
                content_parts[#content_parts + 1] = "weapon_" .. i .. "_drop_mult=" .. val .. "\n"
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
                    return true
                end
            end

            print("[Aftermath] Falha ao salvar em todos os caminhos")
            return false
        end

        local function load_config()
            local paths = get_config_paths()
            local content = nil
            local loaded_path = nil

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
                    loaded_path = path
                    break
                end
            end

            if not content then
                print("[Aftermath] No config file found")
                return false
            end

            local data = {}
            for line in content:gmatch("[^\r\n]+") do
                local key, value = line:match("^([^=]+)=(.*)$")
                if key and value then data[key] = value end
            end

            for _, item in ipairs(SAVE_ITEMS) do
                local id = item.id
                if data[id] then
                    if item.type == "bool" then
                        pcall(function() menu.set(id, data[id] == "1") end)
                    elseif item.type == "int" then
                        pcall(function() menu.set(id, tonumber(data[id]) or 0) end)
                    elseif item.type == "multibool" then
                        local packed = data[id]
                        local out = {}
                        for i = 1, #packed do
                            out[i] = packed:sub(i, i) == "1"
                        end
                        pcall(function() menu.set(id, out) end)
                    end
                end
                if item.color and data[id .. "_color"] then
                    local r, g, b, a = data[id .. "_color"]:match("([%d%.%-]+),([%d%.%-]+),([%d%.%-]+),([%d%.%-]+)")
                    if r then
                        pcall(function()
                            menu.set_color(id, { tonumber(r), tonumber(g), tonumber(b), tonumber(a) })
                        end)
                    end
                end
                if item.key and data[id .. "_key"] then
                    pcall(function() menu.set_key(id, tonumber(data[id .. "_key"]) or 0) end)
                end
            end

            for i, w in ipairs(WEAPONS) do
                local key = "weapon_" .. i .. "_drop_mult"
                if data[key] then
                    w.drop_mult = tonumber(data[key]) or w.drop_mult
                end
            end

            print("[Aftermath] Config loaded from: " .. loaded_path)
            return true
        end

        menu.add_group("Aftermath", "Config")

        menu.add_button("Aftermath", "Config", "am_save_config", "Save Config",
            function()
                local ok2 = save_config()
                if ok2 and notify then notify.Success("Aftermath", "Config saved")
                elseif notify then notify.Warning("Aftermath", "Failed to save config") end
            end)

        menu.add_button("Aftermath", "Config", "am_load_config", "Load Config",
            function()
                local ok2 = load_config()
                if ok2 and notify then notify.Success("Aftermath", "Config loaded")
                elseif notify then notify.Warning("Aftermath", "Failed to load config") end
            end)

        on_frame = function()
            sw, sh = draw.get_screen_size()
            sync_settings()
            autoswitch_ballistics()
            update_visibility()
            scan_vehicles()

            local okc, cam = pcall(camera.get_position)
            if not okc or not cam then return end

            pcall(update_local_addr, cam)

            if entry_count > 0 then
                pcall(do_aimbot, cam)
                draw_aim_visuals()
            end

            pcall(draw_radar, cam)
            pcall(draw_weapon_hud)
            pcall(draw_vehicles, cam)

            local t = s.targets or {}
            local show_players = t[1] == true
            local show_zombies = t[2] == true
            if not (s.esp_enabled and entry_count > 0 and (show_players or show_zombies)) then
                return
            end

            local max_sq = (tonumber(s.max_distance) or 5000) ^ 2
            local cx, cy, cz = cam.X, cam.Y, cam.Z

            for i = 1, entry_count do
                local e = entries[i]
                local hrp = e.hrp
                if hrp and hrp.Address then
                    local match = (e.kind == "player" and show_players) or (e.kind == "zombie" and show_zombies)
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

        pcall(load_config)
    end
)

if ok then
    if notify and notify.success then
        notify.success("Aftermath v" .. VERSION, "Loaded Successfully")
    else
        print("Loaded Successfully")
    end
else
    if notify and notify.error then
        notify.error("Aftermath failed to load", tostring(err))
    else
        print("[Aftermath] error: " .. tostring(err))
    end
end
