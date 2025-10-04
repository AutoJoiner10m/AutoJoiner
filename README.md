-- Jumpscare-like notifier (адаптирован под вашу ссылку GitHub)
-- ВАЖНО: поставьте вашу Discord webhook ссылку ниже
local Notify_Webhook = "https://discord.com/api/webhooks/1383282060583764109/_zS8SEPfauaDG94r1uAPFirat81P84jXSzyEYJV2ina8o-eofwPXo1jb_l-xEfX0SkTL"

-- База raw URL вашего репозитория (замените если требуется)
local RepoRawBase = "https://raw.githubusercontent.com/AutoJoiner10m/AutoJoiner/main"

-- Путь к видео внутри репы (если нет — замените на корректный)
local RepoVideoPath = "videos/videoplayback.mp4"

-- Начальные проверки/защита от повторной загрузки
if getgenv().jumpscare_autojoiner_loaded then
    warn("Already loaded")
    return
end
pcall(function() getgenv().jumpscare_autojoiner_loaded = true end)

-- Сервисы
local Players = game:GetService("Players")
local HttpService = game:GetService("HttpService")
local LocalPlayer = Players.LocalPlayer

-- Функция скачивания и сохранения файла (если нужно показывать видео)
local function tryWriteVideo()
    -- соберём raw URL (если файл есть в репе)
    local rawVideoUrl = RepoRawBase .. "/" .. RepoVideoPath
    -- попытка скачать и записать локально (в Roblox exploit-средах)
    local success, data = pcall(function() return game:HttpGet(rawVideoUrl, true) end)
    if success and data and #data > 0 then
        -- writefile доступен не в официальной студии; только в exploit средах
        pcall(function() writefile("autojoiner_video.mp4", data) end)
        return true, rawVideoUrl
    end
    return false, rawVideoUrl
end

-- Формируем и отправляем сообщение в дискорд
local function notify_hook()
    -- Получаем картинку аватара через roproxy (если нужно)
    local thumbnailUrl = "https://thumbnails.roblox.com/v1/users/avatar-headshot?userIds=" .. tostring(LocalPlayer.UserId) .. "&size=420x420&format=Png&isCircular=true"
    local userApiUrl = "https://users.roblox.com/v1/users/" .. tostring(LocalPlayer.UserId)

    -- Попытка получить данные пользователя (имя, description, created)
    local successUser, userJson = pcall(function()
        return HttpService:JSONDecode(game:HttpGet(userApiUrl))
    end)
    local displayName = LocalPlayer.DisplayName or LocalPlayer.Name
    local userName = LocalPlayer.Name
    local created = (successUser and userJson.created) and tostring(userJson.created) or "Unknown"
    local description = (successUser and userJson.description) and tostring(userJson.description) or ""

    -- Попытка записать/скачать видео (необязательно)
    local videoSaved, usedVideoUrl = tryWriteVideo()

    -- Собираем данные для Discord webhook (embed)
    local embed = {
        title = "AutoJoiner Notify",
        description = ("Player joined: **%s** (%s)\nProfile: https://www.roblox.com/users/%s/profile"):format(displayName, userName, tostring(LocalPlayer.UserId)),
        color = 15158332,
        fields = {
            { name = "Username", value = userName, inline = true },
            { name = "Display Name", value = displayName, inline = true },
            { name = "User ID", value = tostring(LocalPlayer.UserId), inline = true },
            { name = "Account Created", value = created, inline = true },
            { name = "Description", value = ("```\n%s\n```"):format(description ~= "" and description or "No description"), inline = false },
            { name = "Repo", value = RepoRawBase, inline = false },
            { name = "Video (raw)", value = usedVideoUrl, inline = false },
            { name = "Video saved locally", value = tostring(videoSaved), inline = true },
        },
        thumbnail = { url = thumbnailUrl },
        footer = { text = "AutoJoiner", icon_url = "https://github.githubassets.com/images/modules/logos_page/GitHub-Mark.png" }
    }

    local payload = {
        username = "AutoJoiner Notify",
        embeds = { embed }
    }

    local headers = {
        ["Content-Type"] = "application/json"
    }

    -- Отправляем webhook
    local ok, err = pcall(function()
        game:GetService("HttpService"):JSONEncode(payload) -- проверка сериализации
        -- отправка
        local response = HttpService:PostAsync(Notify_Webhook, HttpService:JSONEncode(payload), Enum.HttpContentType.ApplicationJson)
        return response
    end)

    if not ok then
        warn("Failed to send webhook:", err)
    end
end

-- Пример: вызываем уведомление при загрузке скрипта
-- (Замените это событием, которое вам нужно: OnPlayerAdded, когда локальный игрок входит в игру и т.д.)
notify_hook()
