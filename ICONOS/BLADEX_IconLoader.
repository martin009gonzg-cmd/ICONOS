--[[
    BLADEX_IconLoader.lua
    ─────────────────────────────────────────────────────────────
    Módulo centralizado para cargar y acceder a múltiples
    librerías de iconos (lucide, solar, sfsymbols, gravity, etc.)
    guardadas como archivos sueltos en el repo de GitHub ICONOS.

    USO EN OTROS SCRIPTS (una sola línea):

        local IconLoader = loadstring(game:HttpGet(
            "https://raw.githubusercontent.com/martin009gonzg-cmd/BLADEX-HUD/main/BLADEX_IconLoader.lua"
        ))()

        IconLoader.LoadPackage("lucide")
        local id = IconLoader.Get("lucide", "search")   -- "rbxassetid://123..."

    Para pedir varios paquetes de una vez:
        IconLoader.LoadPackages({"lucide", "solar", "gravity"})

    Fallback en cascada (prueba varios sets hasta encontrar el icono):
        local id = IconLoader.GetAny({"lucide","solar","sfsymbols"}, "settings")
--]]

local HttpService = game:GetService("HttpService")

local IconLoader = {}

-- ══════════════════════════════════════════════
-- CONFIGURACIÓN DEL REPO
-- ══════════════════════════════════════════════
local GITHUB_USER   = "martin009gonzg-cmd"
local GITHUB_REPO   = "ICONOS"
local GITHUB_BRANCH = "main"

-- Ruta dentro del repo para cada paquete registrado.
-- Añade aquí nuevos sets sin tocar el resto del código.
local PACKAGE_PATHS = {
    lucide    = "ICONOS/lucide",
    solar     = "ICONOS/solar",
    sfsymbols = "ICONOS/sfsymbols",
    gravity   = "ICONOS/Gravity",
}

local CACHE_FOLDER = "BLADEX_IconCache"

-- ══════════════════════════════════════════════
-- ESTADO INTERNO
-- ══════════════════════════════════════════════
local Packages = {}   -- [packName] = { [iconName] = "rbxassetid://..." }
local Loading  = {}   -- evita cargas duplicadas simultáneas

-- ══════════════════════════════════════════════
-- HELPERS DE URL
-- ══════════════════════════════════════════════
local function apiListUrl(path)
    return string.format(
        "https://api.github.com/repos/%s/%s/contents/%s?ref=%s",
        GITHUB_USER, GITHUB_REPO, path, GITHUB_BRANCH
    )
end

local function rawFileUrl(path)
    return string.format(
        "https://raw.githubusercontent.com/%s/%s/%s/%s",
        GITHUB_USER, GITHUB_REPO, GITHUB_BRANCH, path
    )
end

-- ══════════════════════════════════════════════
-- HELPERS DE CACHE LOCAL (writefile/readfile)
-- ══════════════════════════════════════════════
local function cachePath(packName)
    return CACHE_FOLDER .. "/" .. packName .. ".json"
end

local function ensureCacheFolder()
    pcall(function()
        if not isfolder(CACHE_FOLDER) then
            makefolder(CACHE_FOLDER)
        end
    end)
end

local function readCache(packName)
    local ok, data = pcall(function()
        if isfile(cachePath(packName)) then
            return HttpService:JSONDecode(readfile(cachePath(packName)))
        end
    end)
    if ok and type(data) == "table" then
        return data
    end
    return nil
end

local function writeCache(packName, tbl)
    pcall(function()
        ensureCacheFolder()
        writefile(cachePath(packName), HttpService:JSONEncode(tbl))
    end)
end

-- ══════════════════════════════════════════════
-- EXTRAER EL ASSETID DE UN ARCHIVO INDIVIDUAL
-- ── Soporta dos formatos de archivo automáticamente ──
--   1) Texto plano:      rbxassetid://123456789
--   2) Lua:               return "rbxassetid://123456789"
-- ══════════════════════════════════════════════
local function extractAssetId(rawContent)
    if not rawContent or rawContent == "" then return nil end

    -- Intento 1: es un chunk de Lua que retorna el string
    local ok, result = pcall(function()
        local chunk = loadstring(rawContent)
        if chunk then return chunk() end
    end)
    if ok and type(result) == "string" and result:match("rbxassetid://") then
        return result
    end

    -- Intento 2: texto plano, extraer con patrón
    local id = rawContent:match("(rbxassetid://%d+)")
    if id then return id end

    -- Intento 3: el archivo es solo el número
    local num = rawContent:match("(%d+)")
    if num then return "rbxassetid://" .. num end

    return nil
end

-- Quita la extensión del nombre de archivo para usarlo como nombre de icono
-- ej: "scan-search.lua" -> "scan-search"
local function stripExtension(filename)
    return filename:gsub("%.%w+$", "")
end

-- ══════════════════════════════════════════════
-- CARGA DE UN PAQUETE (listado + descarga individual)
-- ══════════════════════════════════════════════
function IconLoader.LoadPackage(packName, forceRefresh)
    packName = packName:lower()

    if Packages[packName] and not forceRefresh then
        return Packages[packName] -- ya cargado en esta sesión
    end
    if Loading[packName] then
        repeat task.wait(0.05) until not Loading[packName]
        return Packages[packName]
    end

    local path = PACKAGE_PATHS[packName]
    if not path then
        warn("鈿狅笍 BLADEX_IconLoader: paquete desconocido -> " .. tostring(packName))
        return nil
    end

    Loading[packName] = true

    -- 1) Intentar cache local primero (si no se pide refresh)
    if not forceRefresh then
        local cached = readCache(packName)
        if cached then
            Packages[packName] = cached
            Loading[packName] = false
            return cached
        end
    end

    -- 2) Listar archivos de la carpeta vía API de GitHub
    local iconTable = {}
    local listOk, listResult = pcall(function()
        return game:HttpGetAsync(apiListUrl(path))
    end)

    if listOk then
        local decodeOk, files = pcall(function()
            return HttpService:JSONDecode(listResult)
        end)

        if decodeOk and type(files) == "table" then
            for _, fileInfo in ipairs(files) do
                if fileInfo.type == "file" then
                    local iconName = stripExtension(fileInfo.name)
                    local downloadOk, content = pcall(function()
                        return game:HttpGetAsync(rawFileUrl(path .. "/" .. fileInfo.name))
                    end)
                    if downloadOk then
                        local assetId = extractAssetId(content)
                        if assetId then
                            iconTable[iconName] = assetId
                        end
                    end
                end
            end
        else
            warn("鈿狅笍 BLADEX_IconLoader: no se pudo leer el listado de '" .. packName .. "'")
        end
    else
        warn("鈿狅笍 BLADEX_IconLoader: fallo de red listando paquete '" .. packName .. "'")
    end

    -- 3) Guardar resultado (aunque esté vacío, para no reintentar en loop infinito)
    Packages[packName] = iconTable
    if next(iconTable) then
        writeCache(packName, iconTable)
    end

    Loading[packName] = false
    return iconTable
end

-- Carga varios paquetes de una vez
function IconLoader.LoadPackages(packList)
    for _, name in ipairs(packList) do
        IconLoader.LoadPackage(name)
    end
end

-- ══════════════════════════════════════════════
-- ACCESO A ICONOS
-- ══════════════════════════════════════════════

-- Pide un icono de un paquete específico (carga el paquete si hace falta)
function IconLoader.Get(packName, iconName)
    packName = packName:lower()
    local pack = Packages[packName]
    if not pack then
        pack = IconLoader.LoadPackage(packName)
    end
    if pack then
        return pack[iconName]
    end
    return nil
end

-- Prueba varios paquetes en orden hasta encontrar el icono (fallback en cascada)
function IconLoader.GetAny(packOrder, iconName)
    for _, packName in ipairs(packOrder) do
        local id = IconLoader.Get(packName, iconName)
        if id then return id end
    end
    return nil
end

-- Fuerza re-descarga de un paquete ignorando el cache
function IconLoader.Refresh(packName)
    return IconLoader.LoadPackage(packName, true)
end

-- Registra un paquete nuevo en caliente (para futuros sets sin editar este archivo)
function IconLoader.RegisterPackage(name, repoPath)
    PACKAGE_PATHS[name:lower()] = repoPath
end

-- ══════════════════════════════════════════════
-- EXPONER GLOBALMENTE (para que otros scripts del mismo run lo reutilicen
-- sin volver a descargar este loader ni los paquetes ya cargados)
-- ══════════════════════════════════════════════
_G.BLADEX_IconLoader = IconLoader

return IconLoader
