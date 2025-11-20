<script lang="ts">
    import { onMount } from "svelte";
    import type { Map, Marker, LayerGroup, Icon } from "leaflet";

    // 1. VARIABLES DE ESTADO
    let centros_ccss: any[] = [];
    let selectedFilter: "Todos" | "Hospital" | "Clínica" | "EBAIS" = "Todos";
    let searchTerm: string = "";

    let visibleCentros: any[] = [];
    let isListOpen: boolean = false;

    let mapElement: HTMLDivElement;
    let leafletMap: Map | undefined;
    let markersLayer: LayerGroup | undefined;
    let userMarker: Marker | undefined;

    let customIcons: { [key: string]: Icon } = {};
    const filterTypes = ["Todos", "Hospital", "Clínica", "EBAIS"];

    // VARIABLE PARA MODO OSCURO
    let isDarkMode: boolean = false;

    // ------------------------------------------------------------------
    // FUNCIÓN CORREGIDA Y DEPURADA PARA ALTERNAR EL MODO OSCURO
    // ------------------------------------------------------------------
    function toggleDarkMode() {
        isDarkMode = !isDarkMode;

        // --- CÓDIGO DE DEPURACIÓN (Mírelo en la consola F12) ---
        console.log("DEBUG: Toggling dark mode to:", isDarkMode);

        // 1. Guardar la preferencia
        localStorage.setItem("darkMode", isDarkMode.toString());

        // 2. APLICAR LA CLASE AL BODY INMEDIATAMENTE
        if (typeof document !== "undefined") {
            document.body.classList.toggle("dark-mode", isDarkMode);
        }
    }

    // 2. FUNCIÓN PARA OBTENER LOS ÍCONOS PERSONALIZADOS
    function getCustomIcons(L: any) {
        const iconSizeReduced = [18, 30];
        const shadowSizeReduced = [30, 30];
        const iconAnchorReduced = [9, 30];
        const popupAnchorReduced = [1, -25];

        const hospitalIcon = new L.Icon({
            iconUrl:
                "https://raw.githubusercontent.com/pointhi/leaflet-color-markers/master/img/marker-icon-red.png",
            shadowUrl:
                "https://cdnjs.cloudflare.com/ajax/libs/leaflet/0.7.7/images/marker-shadow.png",
            iconSize: iconSizeReduced,
            iconAnchor: iconAnchorReduced,
            popupAnchor: popupAnchorReduced,
            shadowSize: shadowSizeReduced,
        });
        const clinicaIcon = new L.Icon({
            iconUrl:
                "https://raw.githubusercontent.com/pointhi/leaflet-color-markers/master/img/marker-icon-blue.png",
            shadowUrl:
                "https://cdnjs.cloudflare.com/ajax/libs/leaflet/0.7.7/images/marker-shadow.png",
            iconSize: iconSizeReduced,
            iconAnchor: iconAnchorReduced,
            popupAnchor: popupAnchorReduced,
            shadowSize: shadowSizeReduced,
        });
        const ebaisIcon = new L.Icon({
            iconUrl:
                "https://raw.githubusercontent.com/pointhi/leaflet-color-markers/master/img/marker-icon-green.png",
            shadowUrl:
                "https://cdnjs.cloudflare.com/ajax/libs/leaflet/0.7.7/images/marker-shadow.png",
            iconSize: iconSizeReduced,
            iconAnchor: iconAnchorReduced,
            popupAnchor: popupAnchorReduced,
            shadowSize: shadowSizeReduced,
        });
        const userLocationIcon = new L.Icon({
            iconUrl:
                "https://raw.githubusercontent.com/pointhi/leaflet-color-markers/master/img/marker-icon-orange.png",
            shadowUrl:
                "https://cdnjs.cloudflare.com/ajax/libs/leaflet/0.7.7/images/marker-shadow.png",
            iconSize: iconSizeReduced,
            iconAnchor: iconAnchorReduced,
            popupAnchor: popupAnchorReduced,
            shadowSize: shadowSizeReduced,
        });

        customIcons = {
            Hospital: hospitalIcon,
            Clínica: clinicaIcon,
            EBAIS: ebaisIcon,
            UserLocation: userLocationIcon,
        };
    }

    // 3. FUNCIÓN PARA GENERAR ENLACES DE NAVEGACIÓN EN HTML
    function generateNavigationLinks(lat: number, lng: number): string {
        const coords = `${lat},${lng}`;
        // Formato estándar y funcional de Google Maps para buscar coordenadas
        const googleMapsUrl = `https://www.google.com/maps/search/?api=1&query=${coords}`;
        const wazeUrl = `https://waze.com/ul?ll=${coords}&navigate=yes`;

        return `
            <div style="margin-top: 10px; display: flex; gap: 10px;">
                <a href="${googleMapsUrl}" target="_blank" style="text-decoration: none;">
                    <button style="
                        background-color: #4285F4; color: white; border: none; 
                        padding: 8px 12px; border-radius: 4px; cursor: pointer;
                        font-weight: bold; font-size: 0.85em;
                    ">
                        Abrir en Google Maps 🚗
                    </button>
                </a>
                <a href="${wazeUrl}" target="_blank" style="text-decoration: none;">
                    <button style="
                        background-color: #33ccff; color: white; border: none; 
                        padding: 8px 12px; border-radius: 4px; cursor: pointer;
                        font-weight: bold; font-size: 0.85em;
                    ">
                        Abrir en Waze 🗺️
                    </button>
                </a>
            </div>
        `;
    }

    // 4. FUNCIÓN PARA DIBUJAR MARCADORES
    function addMarkersToMap(data: any[]) {
        if (!leafletMap || Object.keys(customIcons).length === 0) return;

        if (markersLayer) {
            leafletMap.removeLayer(markersLayer);
        }

        const L = (window as any).L;
        markersLayer = L.layerGroup().addTo(leafletMap);

        data.forEach((centro) => {
            const lat = centro.latitud;
            const lng = centro.longitud;

            const iconToUse =
                customIcons[centro.tipo] || L.Marker.prototype.options.icon;
            const navigationLinks = generateNavigationLinks(lat, lng);

            const popupContent = `
                <div style="max-width: 250px;">
                    <h4 style="margin-bottom: 5px; color: var(--accent-color);">${centro.nombre}</h4>
                    <p style="margin: 0; font-size: 0.9em;"><strong>Tipo:</strong> ${centro.tipo} (${centro.tipo_ccss})</p>
                    <p style="margin: 0; font-size: 0.9em;"><strong>Dirección:</strong> ${centro.direccion}</p>
                    <p style="margin-top: 5px; font-size: 0.9em;">
                        <strong>Contacto:</strong> <a href="tel:${centro.contacto.split("|")[0].trim()}">${centro.contacto}</a>
                    </p>
                    <p style="margin-top: 5px; font-size: 0.8em; color: #555;">Servicios: ${centro.servicios}</p>
                    ${navigationLinks}
                </div>
            `;

            const marker = L.marker([lat, lng], { icon: iconToUse })
                .bindPopup(popupContent)
                .addTo(markersLayer);

            centro.markerInstance = marker;
        });
    }

    // 5. FÓRMULA DE HAVERSINE PARA CÁLCULO DE DISTANCIA
    function calculateDistance(
        lat1: number,
        lon1: number,
        lat2: number,
        lon2: number,
    ): number {
        const R = 6371; // Radio de la Tierra en kilómetros
        const toRad = (value: number) => (value * Math.PI) / 180;

        const dLat = toRad(lat2 - lat1);
        const dLon = toRad(lon2 - lon1);

        const a =
            Math.sin(dLat / 2) * Math.sin(dLat / 2) +
            Math.cos(toRad(lat1)) *
                Math.cos(toRad(lat2)) *
                Math.sin(dLon / 2) *
                Math.sin(dLon / 2);

        const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
        const distance = R * c;
        return distance;
    }

    // 6. LÓGICA REACTIVA DE FILTRADO Y BÚSQUEDA
    $: if (centros_ccss.length > 0 && leafletMap) {
        const lowerCaseSearchTerm = searchTerm.toLowerCase().trim();

        let filteredData = centros_ccss;
        if (selectedFilter !== "Todos") {
            filteredData = centros_ccss.filter(
                (centro) => centro.tipo === selectedFilter,
            );
        }

        if (lowerCaseSearchTerm) {
            filteredData = filteredData.filter(
                (centro) =>
                    centro.nombre.toLowerCase().includes(lowerCaseSearchTerm) ||
                    centro.direccion
                        .toLowerCase()
                        .includes(lowerCaseSearchTerm) ||
                    centro.contacto.toLowerCase().includes(lowerCaseSearchTerm),
            );
        }

        visibleCentros = filteredData;
        addMarkersToMap(visibleCentros);
    }

    // 7. FUNCIÓN PARA CENTRAR EL MAPA DESDE LA LISTA
    function zoomToCenter(centro: any) {
        if (!leafletMap || !centro.markerInstance) return;

        leafletMap.setView([centro.latitud, centro.longitud], 15);

        centro.markerInstance.openPopup();
    }

    // 8. FUNCIÓN PARA UBICAR AL USUARIO
    function findMe() {
        if (!leafletMap) return;

        if (userMarker) {
            leafletMap.removeLayer(userMarker);
            userMarker = undefined;
        }

        leafletMap.locate({
            setView: true,
            maxZoom: 14,
            enableHighAccuracy: true,
        });

        leafletMap.once("locationfound", function (e) {
            const L = (window as any).L;
            const userLat = e.latlng.lat;
            const userLng = e.latlng.lng;
            const orangeIcon = customIcons["UserLocation"];

            userMarker = L.marker(e.latlng, { icon: orangeIcon })
                .addTo(leafletMap)
                .bindPopup("¡Tu Ubicación Actual!")
                .openPopup();

            // CALCULAR Y ORDENAR DISTANCIAS
            centros_ccss = centros_ccss.map((centro) => {
                const distance = calculateDistance(
                    userLat,
                    userLng,
                    centro.latitud,
                    centro.longitud,
                );
                return {
                    ...centro,
                    distanceKm: parseFloat(distance.toFixed(2)),
                };
            });

            // Ordenar los centros por distancia
            centros_ccss.sort((a, b) => {
                if (a.distanceKm === undefined) return 1;
                if (b.distanceKm === undefined) return -1;
                return a.distanceKm - b.distanceKm;
            });

            isListOpen = true;
            setFilter(selectedFilter);
        });

        leafletMap.once("locationerror", function (e) {
            alert(
                "No fue posible obtener su ubicación. Asegúrese de que la geolocalización esté activada y que el permiso haya sido otorgado. (Error: " +
                    e.message +
                    ")",
            );
        });
    }

    // 9. FUNCIÓN PARA INICIALIZAR EL MAPA (Solo OSM Claro)
    function initializeMap(data: any[]) {
        if (!mapElement || leafletMap) return;

        const L = (window as any).L;

        getCustomIcons(L);

        const defaultIcon = new L.Icon({
            iconUrl:
                "https://unpkg.com/leaflet@1.9.4/dist/images/marker-icon.png",
            shadowUrl:
                "https://unpkg.com/leaflet@1.9.4/dist/images/marker-shadow.png",
            iconSize: [25, 41],
            iconAnchor: [12, 41],
            popupAnchor: [1, -34],
            shadowSize: [41, 41],
        });
        L.Marker.prototype.options.icon = defaultIcon;

        // Inicializar el mapa
        leafletMap = L.map(mapElement).setView([9.95, -84.05], 8);

        // Capa base de OpenStreetMap (SOLO ESTE TILE LAYER)
        L.tileLayer("https://tile.openstreetmap.org/{z}/{x}/{y}.png", {
            maxZoom: 19,
            attribution:
                '© <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
        }).addTo(leafletMap);

        // Inicializar la capa de marcadores
        markersLayer = L.layerGroup().addTo(leafletMap);

        leafletMap.invalidateSize();
    }

    onMount(async () => {
        // Cargar Leaflet
        try {
            const L = await import("leaflet");
            (window as any).L = L;
        } catch (e) {
            console.error(
                "Error al cargar Leaflet. ¿Está instalado correctamente?",
                e,
            );
            return;
        }

        // Cargar preferencia de MODO OSCURO desde el localStorage
        const savedMode = localStorage.getItem("darkMode");

        // Aplicar la preferencia de MODO OSCURO en el montaje
        if (savedMode !== null) {
            isDarkMode = savedMode === "true";
        } else {
            // Usar la preferencia del sistema operativo por defecto
            isDarkMode = window.matchMedia(
                "(prefers-color-scheme: dark)",
            ).matches;
        }

        // Aplicar la clase al BODY inmediatamente después de obtener la preferencia
        if (typeof document !== "undefined") {
            document.body.classList.toggle("dark-mode", isDarkMode);
            console.log("DEBUG: Dark mode applied on load:", isDarkMode);
        }

        try {
            const response = await fetch("/datos_ccss.json");
            if (!response.ok) {
                throw new Error(
                    `Error al cargar datos: ${response.statusText}`,
                );
            }
            const data = await response.json();
            centros_ccss = data;

            initializeMap(centros_ccss);
        } catch (error) {
            console.error("Ocurrió un error al procesar el JSON.", error);
        }
    });

    function setFilter(filterType: "Todos" | "Hospital" | "Clínica" | "EBAIS") {
        selectedFilter = filterType;
    }
</script>

<svelte:head>
    <link
        rel="stylesheet"
        href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"
        integrity="sha256-p4NxAoJBhIIN+hmNHrzRCf9tD/miZyoHS5obTRR9BMY="
        crossorigin=""
    />

    <title>Mapa CCSS Interactivo</title>
</svelte:head>

<main class="page-container">
    <h2>🗺️ Mapa Interactivo de Centros de la CCSS</h2>
    <p>
        Utilice la búsqueda, los filtros o su ubicación para encontrar el centro
        de salud.
    </p>

    <div class="controls-wrapper">
        <div class="search-box">
            <input
                type="text"
                placeholder="🔍 Buscar por nombre, dirección o teléfono..."
                bind:value={searchTerm}
            />
        </div>

        <div class="filter-controls">
            <button
                class="theme-toggle-button"
                on:click={toggleDarkMode}
                title="Alternar entre modo claro y modo oscuro"
            >
                {isDarkMode ? "🌞 Modo Claro" : "🌙 Modo Oscuro"}
            </button>

            <button
                class="filter-button location-button"
                on:click={findMe}
                title="Centrar mapa en su ubicación actual y calcular distancias"
            >
                📍 Mi Ubicación
            </button>

            <button
                class="filter-button list-toggle-button"
                on:click={() => (isListOpen = !isListOpen)}
            >
                {isListOpen
                    ? "Cerrar Lista ✖️"
                    : `Ver Lista (${visibleCentros.length}) ☰`}
            </button>

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

    <div class="map-and-list-container" class:list-closed={!isListOpen}>
        <div id="map" bind:this={mapElement}></div>

        {#if isListOpen}
            <div class="results-list">
                <div class="list-header">
                    <h3>Resultados ({visibleCentros.length})</h3>
                    {#if centros_ccss[0] && centros_ccss[0].distanceKm !== undefined}
                        <p class="list-sorted-info">
                            Ordenado por **Distancia**.
                        </p>
                    {/if}
                </div>

                {#if visibleCentros.length > 0}
                    {#each visibleCentros as centro}
                        <div
                            class="center-item"
                            on:click={() => zoomToCenter(centro)}
                        >
                            <p class="center-name">
                                {centro.nombre}
                                <span
                                    class="center-type"
                                    class:hospital={centro.tipo === "Hospital"}
                                    class:clinica={centro.tipo === "Clínica"}
                                    class:ebais={centro.tipo === "EBAIS"}
                                >
                                    {centro.tipo}
                                </span>
                            </p>

                            {#if centro.distanceKm !== undefined}
                                <p class="center-distance">
                                    Distancia: **{centro.distanceKm} km**
                                </p>
                            {/if}

                            <p class="center-address">📍 {centro.direccion}</p>
                            <p class="center-contact">
                                📞 {centro.contacto.split("|")[0].trim()}
                            </p>

                            <div class="navigation-links-list">
                                <a
                                    href={`https://www.google.com/maps/search/?api=1&query=${centro.latitud},${centro.longitud}`}
                                    target="_blank"
                                    class="nav-button google-maps"
                                >
                                    Google Maps 🚗
                                </a>
                                <a
                                    href={`https://waze.com/ul?ll=${centro.latitud},${centro.longitud}&navigate=yes`}
                                    target="_blank"
                                    class="nav-button waze"
                                >
                                    Waze 🗺️
                                </a>
                            </div>
                        </div>
                    {/each}
                {:else}
                    <p class="no-results">
                        No se encontraron centros con los filtros aplicados.
                    </p>
                {/if}
            </div>
        {/if}
    </div>

    {#if centros_ccss.length === 0}
        <p style="text-align: center; color: red;">
            Cargando datos o no se encontraron datos. Verifique la consola para
            errores.
        </p>
    {/if}
</main>

<style>
    /* ----------------------------------------------------- */
    /* VARIABLES CSS PARA MODO CLARO Y OSCURO (DEFINICIÓN FINAL) */
    /* ----------------------------------------------------- */

    /* MODO CLARO (Aplicado al body por defecto) */
    :global(body) {
        /* Colores Base (Claro) */
        --bg-color: #ffffff;
        --text-color: #333333;
        --panel-bg: #f9f9f9;
        --border-color: #cccccc;
        --accent-color: #004a8b; /* Azul CCSS */
        --accent-color-hover: #003663;
        --secondary-color: #f7931e; /* Naranja Ubicación */
        --list-item-hover: #e6f0f8;
        --list-item-border: #eeeeee;
        --shadow-color: rgba(0, 0, 0, 0.1);

        /* Estilos del body que usan las variables */
        background-color: var(--bg-color);
        color: var(--text-color);
        transition:
            background-color 0.3s,
            color 0.3s;
    }

    /* MODO OSCURO (Anula variables al tener la clase dark-mode) */
    :global(body.dark-mode) {
        --bg-color: #121212;
        --text-color: #f5f5f5;
        --panel-bg: #1e1e1e;
        --border-color: #333333;
        --accent-color: #42a5f5; /* Azul CCSS más brillante */
        --accent-color-hover: #2196f3;
        --secondary-color: #ffb300; /* Naranja Ubicación brillante */
        --list-item-hover: #2a2a2a;
        --list-item-border: #444444;
        --shadow-color: rgba(255, 255, 255, 0.1);
    }

    /* ----------------------------------------------------- */
    /* ESTILOS GENERALES APLICANDO VARIABLES */
    /* ----------------------------------------------------- */

    .page-container {
        padding: 10px;
        max-width: 100%;
        margin: 0 auto;
    }

    h2 {
        margin-top: 0;
        margin-bottom: 5px;
        font-size: 1.8em;
        color: var(--accent-color);
        border-bottom: 2px solid var(--accent-color);
        padding-bottom: 10px;
    }

    .controls-wrapper {
        display: flex;
        flex-direction: column;
        gap: 10px;
        margin-bottom: 15px;
    }

    .map-and-list-container {
        display: flex;
        gap: 20px;
        margin-top: 5px;
    }

    #map {
        flex-grow: 1;
        height: 95vh;
        width: 70%;
        border-radius: 8px;
        box-shadow: 0 4px 12px var(--shadow-color);
        margin-top: 5px;
        transition: width 0.3s ease;
    }

    .map-and-list-container.list-closed #map {
        width: 100%;
    }

    .search-box input {
        width: 100%;
        padding: 12px 15px;
        font-size: 16px;
        border: 2px solid var(--border-color);
        border-radius: 8px;
        box-sizing: border-box;
        transition: border-color 0.2s;
        background-color: var(--bg-color);
        color: var(--text-color);
    }
    .search-box input:focus {
        border-color: var(--accent-color);
        outline: none;
    }

    .filter-controls {
        display: flex;
        gap: 10px;
        flex-wrap: wrap;
    }

    .theme-toggle-button {
        padding: 10px 15px;
        border: 2px solid var(--text-color);
        border-radius: 5px;
        background-color: transparent;
        color: var(--text-color);
        font-weight: bold;
        cursor: pointer;
        transition: all 0.2s ease;
        order: -2;
    }

    .theme-toggle-button:hover {
        background-color: var(--accent-color);
        color: var(--bg-color);
        border-color: var(--accent-color);
    }

    .filter-button {
        padding: 10px 15px;
        border: 2px solid var(--accent-color);
        border-radius: 5px;
        background-color: var(--bg-color);
        color: var(--accent-color);
        font-weight: bold;
        cursor: pointer;
        transition: all 0.2s ease;
    }

    .filter-button:hover:not(.active) {
        background-color: var(--list-item-hover);
    }

    .filter-button.location-button {
        border-color: var(--secondary-color);
        color: var(--secondary-color);
        order: -1;
    }

    .filter-button.location-button:hover {
        background-color: var(--secondary-color);
        color: var(--bg-color);
    }

    .filter-button.list-toggle-button {
        background-color: var(--panel-bg);
        color: var(--text-color);
        border-color: var(--border-color);
    }
    .filter-button.list-toggle-button:hover {
        background-color: var(--list-item-hover);
        border-color: var(--border-color);
    }

    .filter-button.active {
        background-color: var(--accent-color);
        color: white;
        box-shadow: 0 2px 5px var(--shadow-color);
    }

    .results-list {
        width: 300px;
        max-height: 95vh;
        min-width: 300px;
        overflow-y: auto;
        border: 1px solid var(--border-color);
        border-radius: 8px;
        box-shadow: 0 4px 12px var(--shadow-color);
        padding: 10px;
        background-color: var(--panel-bg);
    }

    .list-header h3 {
        margin: 0;
        color: var(--accent-color);
    }

    .list-sorted-info {
        font-size: 0.75em;
        color: var(--secondary-color);
        margin: 0 0 10px 0;
        padding-bottom: 10px;
        border-bottom: 1px solid var(--list-item-border);
    }

    .center-item {
        padding: 10px 0;
        border-bottom: 1px dashed var(--list-item-border);
        cursor: pointer;
        transition: background-color 0.1s;
        position: relative;
    }

    .center-item:hover {
        background-color: var(--list-item-hover);
    }

    .center-item > *:not(.navigation-links-list) {
        pointer-events: none;
    }

    .center-name {
        font-weight: bold;
        margin: 0 0 4px 0;
        display: flex;
        justify-content: space-between;
        align-items: center;
        color: var(--text-color);
    }

    .center-distance {
        font-size: 0.9em;
        color: var(--accent-color);
        margin: 4px 0;
        font-weight: bold;
        pointer-events: none;
    }

    .center-type {
        font-size: 0.75em;
        font-weight: normal;
        padding: 3px 6px;
        border-radius: 4px;
        color: white;
    }

    .center-type.hospital {
        background-color: #d9534f;
    }
    .center-type.clinica {
        background-color: #42a5f5;
    }
    .center-type.ebais {
        background-color: #5cb85c;
    }

    .center-address,
    .center-contact {
        font-size: 0.85em;
        color: var(--text-color);
        opacity: 0.7;
        margin: 0;
        pointer-events: none;
    }

    /* ESTILOS DE NAVEGACIÓN EN LA LISTA */
    .navigation-links-list {
        display: flex;
        gap: 8px;
        margin-top: 8px;
        pointer-events: all;
    }

    .nav-button {
        text-decoration: none;
        padding: 6px 10px;
        border-radius: 4px;
        font-size: 0.75em;
        font-weight: bold;
        transition: background-color 0.1s;
        text-align: center;
        flex-grow: 1;
    }

    .nav-button.google-maps {
        background-color: #4285f4;
        color: white;
        border: 1px solid #4285f4;
    }
    .nav-button.google-maps:hover {
        background-color: #357ae8;
    }

    .nav-button.waze {
        background-color: #33ccff;
        color: white;
        border: 1px solid #33ccff;
    }
    .nav-button.waze:hover {
        background-color: #24a4d2;
    }

    /* Media query para dispositivos pequeños */
    @media (max-width: 1000px) {
        .map-and-list-container {
            flex-direction: column;
        }
        #map {
            width: 100%;
            height: 60vh;
        }

        .map-and-list-container.list-closed #map {
            width: 100%;
        }

        .results-list {
            width: 100%;
            max-height: 40vh;
        }
    }

    /* ----------------------------------------------------- */
    /* ESTILOS DE MODO OSCURO PARA ELEMENTOS DE LEAFLET (POPUPs y CONTROLES) */
    /* ----------------------------------------------------- */

    /* NOTA: Usamos :global(...) en todo el selector para asegurar la aplicación al DOM global de Leaflet. */

    /* Controles de Zoom */
    :global(body.dark-mode .leaflet-control-zoom) {
        border: 1px solid var(--border-color);
    }

    /* Fondo del Popup y Tip (Flecha) */
    :global(body.dark-mode .leaflet-popup-content-wrapper),
    :global(body.dark-mode .leaflet-popup-tip) {
        background: var(--panel-bg);
        color: var(--text-color);
        box-shadow: 0 3px 14px var(--shadow-color);
    }

    /* Asegurar que el título del popup sea del color acentuado */
    :global(body.dark-mode .leaflet-popup-content h4) {
        color: var(--accent-color) !important;
    }

    /* Tooltip */
    :global(body.dark-mode .leaflet-tooltip) {
        background: var(--panel-bg);
        color: var(--text-color);
        border: 1px solid var(--border-color);
    }

    /* Botón de cerrar popup */
    :global(body.dark-mode .leaflet-popup-close-button) {
        color: var(--text-color);
        opacity: 0.8;
    }
    :global(body.dark-mode .leaflet-popup-close-button):hover {
        color: var(--accent-color);
    }
</style>
