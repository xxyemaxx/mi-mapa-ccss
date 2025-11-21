<script lang="ts">
    import { onMount } from "svelte";
    import type { Map, Marker, Icon, TileLayer } from "leaflet";

    // 1. VARIABLES DE ESTADO
    let centros_ccss: any[] = [];
    let selectedFilter: "Todos" | "Hospital" | "Clínica" | "EBAIS" = "Todos";
    let searchTerm: string = "";

    // Variable reactiva para almacenar los centros filtrados
    let filteredCentros: any[] = [];

    let mapElement: HTMLDivElement;
    let leafletMap: Map | undefined;

    // El tipo es 'any' ya que L.markerClusterGroup no es un tipo estándar de Leaflet
    let markersLayer: any | undefined;

    let userMarker: Marker | undefined;
    let customIcons: { [key: string]: Icon } = {};

    let isDarkMode: boolean = false;

    const filterTypes = ["Todos", "Hospital", "Clínica", "EBAIS"];
    const validCenterTypes = ["Hospital", "Clínica", "EBAIS"];

    // FUNCIÓN PARA ALTERNAR EL MODO OSCURO (sin cambios)
    function toggleDarkMode() {
        isDarkMode = !isDarkMode;
        localStorage.setItem("darkMode", isDarkMode.toString());

        if (typeof document !== "undefined") {
            document.body.classList.toggle("dark-mode", isDarkMode);
            leafletMap?.invalidateSize();
        }
    }

    // 2. FUNCIÓN PARA OBTENER EL ÍCONO (Tamaño pequeño)
    function getCustomIcon(type: string): Icon {
        const L = (window as any).L;
        const typeKey = validCenterTypes.includes(type) ? type : "Default";

        if (customIcons[typeKey]) {
            return customIcons[typeKey];
        }

        let colorClass = "blue";
        if (type === "Hospital") colorClass = "red";
        else if (type === "Clínica") colorClass = "blue";
        else if (type === "EBAIS") colorClass = "green";

        const newIcon = L.icon({
            iconUrl: `https://raw.githubusercontent.com/pointhi/leaflet-color-markers/master/img/marker-icon-2x-${colorClass}.png`,
            shadowUrl:
                "https://cdnjs.cloudflare.com/ajax/libs/leaflet/0.7.7/images/marker-shadow.png",
            iconSize: [20, 33], // Tamaño del ícono: Reducido
            iconAnchor: [10, 33], // Punto de anclaje: Ajustado
            popupAnchor: [1, -28], // Posición del popup: Ajustado
            shadowSize: [33, 33], // Tamaño de la sombra: Ajustado
        });

        customIcons[typeKey] = newIcon;
        return newIcon;
    }

    // 3. FUNCIÓN PARA DIBUJAR MARCADORES (ACTUALIZADA: URLs de ruta corregidas)
    function addMarkersToMap(data: any[]) {
        if (!leafletMap || !markersLayer) return;

        markersLayer.clearLayers();

        const L = (window as any).L;

        // OBTENER COORDENADAS DEL USUARIO
        const userLat = userMarker ? userMarker.getLatLng().lat : null;
        const userLng = userMarker ? userMarker.getLatLng().lng : null;
        const userCoordsParam =
            userLat !== null && userLng !== null ? `${userLat},${userLng}` : "";

        data.forEach((centro) => {
            const lat = centro.latitud;
            const lng = centro.longitud;
            const destCoords = `${lat},${lng}`;

            const icon = getCustomIcon(centro.tipo);

            // CONSTRUCCIÓN DE LINKS DINÁMICOS
            let googleMapsUrl: string;
            let wazeUrl: string;
            let routingInfo = "";

            if (userCoordsParam) {
                // CORRECCIÓN GOOGLE MAPS: Usar el formato estándar de Google Maps API para ruta (origin/destination)
                googleMapsUrl = `https://www.google.com/maps/dir/?api=1&origin=${userCoordsParam}&destination=${destCoords}&travelmode=driving`;

                // WAZE: Sintaxis correcta para origen y destino
                wazeUrl = `https://waze.com/ul?ll=${destCoords}&navigate=yes&from_coord=${userCoordsParam}`;

                routingInfo =
                    '<p style="margin: 0; font-size: 0.8em; font-weight: bold; color: var(--secondary-color);">¡Ruta calculada desde tu ubicación! (Distancia/Tiempo en la app)</p>';
            } else {
                // Links de DESTINO (Fallback)
                // Google Maps: Solo destino (la app intentará usar la ubicación del dispositivo como origen)
                googleMapsUrl = `https://www.google.com/maps/search/?api=1&query=${destCoords}`;
                wazeUrl = `https://waze.com/ul?ll=${destCoords}&navigate=yes`;

                routingInfo =
                    '<p style="margin: 0; font-size: 0.8em; color: #888;">Para calcular la ruta automáticamente, presiona "📍 Mostrar mi Ubicación" primero.</p>';
            }

            const popupContent = `
                <div style="max-width: 250px;">
                    <h4 style="margin-bottom: 5px; color: var(--accent-color);">${centro.nombre}</h4>
                    ${routingInfo} 
                    <p style="margin: 0; font-size: 0.9em;"><strong>Tipo:</strong> ${centro.tipo} (${centro.tipo_ccss})</p>
                    <p style="margin: 0; font-size: 0.9em;"><strong>Dirección:</strong> ${centro.direccion}</p>
                    <p style="margin-top: 5px; font-size: 0.9em;">
                        <strong>Contacto:</strong> <a href="tel:${centro.contacto.split("|")[0].trim()}">${centro.contacto}</a>
                    </p>
                    <p style="margin-top: 5px; font-size: 0.8em; color: #555;">Servicios: ${centro.servicios}</p>
                    
                    <div style="margin-top: 10px; display: flex; gap: 10px;">
                        <a href="${googleMapsUrl}" target="_blank">Google Maps 🚗</a>
                        <a href="${wazeUrl}" target="_blank">Waze 🗺️</a>
                    </div>
                </div>
            `;

            L.marker([lat, lng], { icon: icon })
                .bindPopup(popupContent)
                .addTo(markersLayer);
        });
    }

    // 4. FUNCIÓN PARA BUSCAR LA UBICACIÓN DEL USUARIO (sin cambios funcionales)
    function locateUser() {
        if (!leafletMap) return;

        alert("Buscando tu ubicación...");

        leafletMap.locate({ setView: true, maxZoom: 14 });

        const L = (window as any).L;

        leafletMap.once("locationfound", function (e) {
            if (userMarker) {
                leafletMap.removeLayer(userMarker);
            }

            const userIcon = L.circleMarker(e.latlng, {
                radius: 8,
                fillColor: "#0078FF",
                color: "#000",
                weight: 1,
                opacity: 1,
                fillOpacity: 0.8,
            });

            userMarker = userIcon.addTo(leafletMap);
            userMarker.bindPopup("¡Estás Aquí!").openPopup();

            // Forzar la re-ejecución del filtro reactivo para actualizar los popups
            if (centros_ccss.length > 0) {
                // Pequeño truco para forzar el re-render de Svelte
                searchTerm = searchTerm.trim() + " ";
                searchTerm = searchTerm.trim();
            }
        });

        leafletMap.once("locationerror", function (e) {
            alert("No fue posible obtener tu ubicación. " + e.message);
        });
    }

    // 5. LÓGICA REACTIVA DE FILTRADO Y BÚSQUEDA (Estable y refactorizada)

    // BLOQUE 5a: Filtra los datos basado en las variables reactivas (filtro y búsqueda)
    $: {
        if (centros_ccss.length === 0) {
            filteredCentros = [];
        } else {
            const lowerCaseSearchTerm = searchTerm.toLowerCase().trim();

            let data = centros_ccss;

            // 1. FILTRO POR TIPO
            if (selectedFilter !== "Todos") {
                data = centros_ccss.filter(
                    (centro) => centro.tipo === selectedFilter,
                );
            }

            // 2. FILTRO POR BÚSQUEDA
            if (lowerCaseSearchTerm) {
                data = data.filter(
                    (centro) =>
                        centro.nombre
                            .toLowerCase()
                            .includes(lowerCaseSearchTerm) ||
                        centro.direccion
                            .toLowerCase()
                            .includes(lowerCaseSearchTerm) ||
                        centro.contacto
                            .toLowerCase()
                            .includes(lowerCaseSearchTerm),
                );
            }

            filteredCentros = data;
        }
    }

    // BLOQUE 5b: Dibuja los marcadores cada vez que la lista filtrada cambia O el mapa está listo
    $: if (filteredCentros && leafletMap) {
        addMarkersToMap(filteredCentros);
    }

    // 6. FUNCIÓN PARA CAMBIAR EL FILTRO DE TIPO (sin cambios)
    function setFilter(filterType: "Todos" | "Hospital" | "Clínica" | "EBAIS") {
        selectedFilter = filterType;
    }

    // 7. FUNCIÓN PARA INICIALIZAR EL MAPA (sin cambios funcionales)
    function initializeMap(data: any[]) {
        if (!mapElement || leafletMap) return;

        const L = (window as any).L;

        // Inicializar el mapa
        leafletMap = L.map(mapElement).setView([9.95, -84.05], 8);

        // --- LÓGICA DE CONTROL DE CAPAS ---
        const osmStandard: TileLayer = L.tileLayer(
            "https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png",
            {
                maxZoom: 19,
                attribution:
                    '© <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
            },
        );
        const topoMap: TileLayer = L.tileLayer(
            "https://{s}.tile.opentopomap.org/{z}/{x}/{y}.png",
            {
                maxZoom: 17,
                attribution:
                    'Map data: © <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors, <a href="http://viewfinderpanoramas.org">SRTM</a> | Map style: © <a href="https://opentopomap.org">OpenTopoMap</a> (<a href="https://creativecommons.org/licenses/by-sa/3.0/">CC-BY-SA</a>)',
            },
        );
        const humanitarian: TileLayer = L.tileLayer(
            "http://{s}.tile.openstreetmap.fr/hot/{z}/{x}/{y}.png",
            {
                maxZoom: 19,
                attribution:
                    '© <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors, Tiles courtesy of <a href="http://hot.openstreetmap.org/" target="_blank">Humanitarian OpenStreetMap Team</a>',
            },
        );

        const baseLayers = {
            "1. OSM Estándar": osmStandard,
            "2. Topografía (OpenTopo)": topoMap,
            "3. Humanitario (HOT)": humanitarian,
        };

        let layerName = "Centros CCSS (Marcadores)";

        // VERIFICACIÓN Y FALLBACK DE CLUSTERIZACIÓN
        if (typeof L.markerClusterGroup === "function") {
            markersLayer = L.markerClusterGroup();
            layerName = "Centros CCSS (CLUSTERS)";
            console.log("DEBUG: MarkerClusterGroup inicializado.");
        } else {
            // FALLBACK SEGURO: Usar capa de grupo regular si el plugin no carga
            markersLayer = L.layerGroup();
            console.error(
                "DEBUG: MarkerClusterGroup no encontrado. Usando L.layerGroup().",
            );
        }

        // La capa de marcadores debe agregarse al mapa
        markersLayer.addTo(leafletMap);

        const overlayLayers = {
            [layerName]: markersLayer,
        };

        osmStandard.addTo(leafletMap);
        L.control
            .layers(baseLayers, overlayLayers, { collapsed: false })
            .addTo(leafletMap);
    }

    onMount(async () => {
        // Cargar Leaflet
        const L = await import("leaflet");
        (window as any).L = L;

        // Ajustar el ícono por defecto (IMPORTANTE para el funcionamiento de marcadores)
        const defaultIcon = new L.Icon({
            iconUrl:
                "https://unpkg.com/leaflet@1.9.4/dist/images/marker-icon.png",
            shadowUrl:
                "https://unpkg.com/leaflet@1.9.4/dist/images/marker-shadow.png",
            iconSize: [20, 33],
            iconAnchor: [10, 33],
            popupAnchor: [1, -28],
            shadowSize: [33, 33],
        });
        L.Marker.prototype.options.icon = defaultIcon;

        // INICIALIZACIÓN DEL MODO OSCURO
        const savedMode = localStorage.getItem("darkMode");
        if (savedMode !== null) {
            isDarkMode = savedMode === "true";
        } else {
            isDarkMode = window.matchMedia(
                "(prefers-color-scheme: dark)",
            ).matches;
        }
        if (typeof document !== "undefined") {
            document.body.classList.toggle("dark-mode", isDarkMode);
        }
        // FIN INICIALIZACIÓN MODO OSCURO

        try {
            const response = await fetch("/datos_ccss.json");
            if (!response.ok) {
                throw new Error(
                    `Error al cargar datos: ${response.statusText}`,
                );
            }
            const data = await response.json();
            centros_ccss = data;
            filteredCentros = data; // Inicializar la lista filtrada
            initializeMap(centros_ccss);
        } catch (error) {
            console.error("Ocurrió un error al procesar el JSON.", error);
        }
    });
</script>

<svelte:head>
    <link
        rel="stylesheet"
        href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"
        integrity="sha256-p4NxAoJBhIIN+hmNHrzRCf9tD/miZyoHS5obTRR9BMY="
        crossorigin=""
    />
    <link
        rel="stylesheet"
        href="https://cdn.jsdelivr.net/npm/leaflet-control-layers-compat@1.1.0/control.layers.css"
    />

    <link
        rel="stylesheet"
        href="https://unpkg.com/leaflet.markercluster@1.4.1/dist/MarkerCluster.css"
    />
    <link
        rel="stylesheet"
        href="https://unpkg.com/leaflet.markercluster@1.4.1/dist/MarkerCluster.Default.css"
    />

    <script
        src="https://unpkg.com/leaflet.markercluster@1.4.1/dist/leaflet.markercluster.js"
    ></script>

    <title>Mapa CCSS Interactivo - Cluster</title>
</svelte:head>

<main class="osm-layout-container">
    <div class="sidebar">
        <h3 class="sidebar-title">🏥 Centros CCSS</h3>

        <div class="control-group">
            <div class="search-box">
                <input
                    type="text"
                    placeholder="🔍 Buscar centro..."
                    bind:value={searchTerm}
                />
            </div>
        </div>

        <div class="control-group theme-toggle-section">
            <button class="theme-toggle-button" on:click={toggleDarkMode}>
                {isDarkMode ? "🌞 Modo Claro" : "🌙 Modo Oscuro"}
            </button>
        </div>

        <div class="control-group locate-section">
            <button class="locate-button" on:click={locateUser}>
                <span style="font-size: 1.2em; vertical-align: middle;">📍</span
                > Mostrar mi Ubicación
            </button>
        </div>

        <div class="control-group">
            <p class="control-label">Filtrar por Tipo:</p>
            <div class="filter-controls">
                {#each filterTypes as type}
                    <button
                        class="filter-button"
                        class:active={selectedFilter === type}
                        on:click={() => setFilter(type)}
                    >
                        {type}
                    </button>
                {/each}
            </div>
        </div>

        {#if centros_ccss.length === 0}
            <p class="loading-message">Cargando datos...</p>
        {/if}
    </div>

    <div id="map" bind:this={mapElement}></div>
</main>

<style>
    /* ----------------------------------------------------- */
    /* VARIABLES CSS PARA MODO CLARO Y OSCURO */
    /* ----------------------------------------------------- */

    /* MODO CLARO (Aplicado al body por defecto) */
    :global(body) {
        /* Colores Base (Claro) */
        --bg-color: #ffffff; /* Fondo principal de la página */
        --text-color: #333333; /* Color de texto principal */
        --sidebar-bg: #f8f9fa; /* Fondo de la barra lateral */
        --border-color: #cccccc; /* Color de bordes y separadores */
        --accent-color: #004a8b; /* Azul CCSS */
        --accent-color-hover: #003663;
        --secondary-color: #28a745; /* Verde para ubicación/botones */
        --shadow-color: rgba(0, 0, 0, 0.1);

        /* Estilos del body que usan las variables */
        background-color: var(--bg-color);
        color: var(--text-color);
        transition:
            background-color 0.3s,
            color 0.3s;
        height: 100vh;
        overflow: hidden;
        font-family: "Helvetica Neue", Arial, sans-serif;
    }

    /* MODO OSCURO (Anula variables al tener la clase dark-mode) */
    :global(body.dark-mode) {
        --bg-color: #121212;
        --text-color: #f5f5f5;
        --sidebar-bg: #1e1e1e;
        --border-color: #333333;
        --accent-color: #42a5f5; /* Azul CCSS más brillante */
        --accent-color-hover: #2196f3;
        --secondary-color: #5cb85c; /* Verde brillante */
        --shadow-color: rgba(255, 255, 255, 0.1);
    }

    /* ----------------------------------------------------- */
    /* ESTILOS DE LAYOUT Y BARRA LATERAL (sin cambios) */
    /* ----------------------------------------------------- */

    .osm-layout-container {
        display: flex;
        height: 100vh;
        width: 100vw;
    }

    .sidebar {
        width: 300px;
        background-color: var(--sidebar-bg);
        padding: 15px;
        box-shadow: 2px 0 5px var(--shadow-color);
        z-index: 1000;
        overflow-y: auto;
        border-right: 1px solid var(--border-color);
    }

    .sidebar-title {
        color: var(--accent-color);
        margin-top: 0;
        margin-bottom: 20px;
        border-bottom: 2px solid var(--accent-color);
        padding-bottom: 10px;
        font-size: 1.5em;
    }

    #map {
        flex-grow: 1;
        height: 100%;
        width: 100%;
        min-width: 0;
        z-index: 1;
    }

    .control-group {
        margin-bottom: 20px;
        padding-bottom: 10px;
        border-bottom: 1px solid var(--border-color);
    }

    .locate-section,
    .theme-toggle-section {
        border-bottom: none;
    }

    /* Botón de Modo Oscuro */
    .theme-toggle-button {
        width: 100%;
        padding: 10px;
        background-color: var(--accent-color);
        color: white;
        border: 1px solid var(--accent-color);
        border-radius: 4px;
        font-size: 1em;
        font-weight: bold;
        cursor: pointer;
        transition: background-color 0.2s;
    }
    .theme-toggle-button:hover {
        background-color: var(--accent-color-hover);
        border-color: var(--accent-color-hover);
    }

    /* Botón de Ubicación */
    .locate-button {
        width: 100%;
        padding: 10px;
        background-color: var(--secondary-color);
        color: white;
        border: none;
        border-radius: 4px;
        font-size: 1em;
        font-weight: bold;
        cursor: pointer;
        transition: background-color 0.2s;
    }
    .locate-button:hover {
        background-color: var(--accent-color-hover);
    }

    .control-label {
        font-weight: bold;
        margin-bottom: 5px;
        color: var(--text-color);
    }

    .search-box input {
        width: 100%;
        padding: 10px;
        font-size: 15px;
        border: 1px solid var(--border-color);
        border-radius: 4px;
        box-sizing: border-box;
        background-color: var(--bg-color);
        color: var(--text-color);
    }
    .search-box input:focus {
        border-color: var(--accent-color);
        outline: none;
        box-shadow: 0 0 5px var(--shadow-color);
    }

    .filter-controls {
        display: flex;
        flex-wrap: wrap;
        gap: 8px;
    }

    .filter-button {
        padding: 8px 12px;
        border: 1px solid var(--accent-color);
        border-radius: 4px;
        background-color: var(--bg-color);
        color: var(--accent-color);
        font-weight: 500;
        cursor: pointer;
        transition: all 0.15s ease;
        font-size: 0.9em;
    }

    .filter-button.active {
        background-color: var(--accent-color);
        color: white;
        box-shadow: 0 1px 3px var(--shadow-color);
    }

    .filter-button:hover:not(.active) {
        background-color: var(--sidebar-bg);
    }

    .loading-message {
        text-align: center;
        color: #888;
        font-style: italic;
        padding: 20px 0;
    }

    /* ----------------------------------------------------- */
    /* ESTILOS DE MODO OSCURO PARA LEAFLET */
    /* ----------------------------------------------------- */

    /* Controles de Zoom */
    :global(body.dark-mode .leaflet-control-zoom a) {
        background-color: var(--sidebar-bg);
        border-bottom: 1px solid var(--border-color);
        color: var(--text-color) !important;
    }

    /* Estilos del Control de Capas en Modo Oscuro */
    :global(body.dark-mode .leaflet-control-layers) {
        background: var(--sidebar-bg);
        color: var(--text-color);
        border: 1px solid var(--border-color);
        box-shadow: 0 1px 5px var(--shadow-color);
    }
    :global(body.dark-mode .leaflet-control-layers-base),
    :global(body.dark-mode .leaflet-control-layers-overlays) {
        color: var(--text-color);
    }
    :global(body.dark-mode .leaflet-control-layers-toggle) {
        filter: invert(100%);
    }

    /* Fondo del Popup y Tip (Flecha) */
    :global(body.dark-mode .leaflet-popup-content-wrapper),
    :global(body.dark-mode .leaflet-popup-tip) {
        background: var(--sidebar-bg);
        color: var(--text-color);
        box-shadow: 0 3px 14px var(--shadow-color);
    }

    /* Título y elementos del Popup */
    :global(.leaflet-popup-content h4) {
        color: var(--accent-color) !important;
    }
    :global(body.dark-mode .leaflet-popup-content p) {
        color: var(--text-color);
    }
    :global(body.dark-mode .leaflet-popup-close-button) {
        color: var(--text-color);
    }

    /* Estilos de navegación en el popup */
    :global(.leaflet-popup-content a) {
        text-decoration: none;
        color: white;
        padding: 4px 8px;
        border-radius: 4px;
        font-weight: bold;
        font-size: 0.8em;
    }
    :global(.leaflet-popup-content a[href*="google.com"]) {
        background-color: #4285f4;
    }
    :global(.leaflet-popup-content a[href*="waze.com"]) {
        background-color: #33ccff;
    }

    /* Estilos para la apariencia de los clusters en Modo Oscuro */
    :global(body.dark-mode .marker-cluster-small) {
        background-color: rgba(66, 165, 245, 0.6);
    }
    :global(body.dark-mode .marker-cluster-medium) {
        background-color: rgba(33, 150, 243, 0.6);
    }
    :global(body.dark-mode .marker-cluster-large) {
        background-color: rgba(21, 101, 192, 0.6);
    }
    :global(body.dark-mode .marker-cluster-small div),
    :global(body.dark-mode .marker-cluster-medium div),
    :global(body.dark-mode .marker-cluster-large div) {
        background-color: rgba(66, 165, 245, 0.9);
        color: #121212;
    }
</style>
