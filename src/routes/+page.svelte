<script lang="ts">
    import { onMount } from "svelte";
    import type {
        Map,
        Marker,
        Icon,
        TileLayer,
        LatLngBoundsExpression,
        LatLngExpression,
    } from "leaflet";

    // 1. VARIABLES DE ESTADO
    let centros_ccss: any[] = [];
    let selectedFilter: "Todos" | "Hospital" | "Clínica" | "EBAIS" = "Todos";
    let searchTerm: string = "";

    // Contendrá los centros filtrados, con la distancia calculada y ordenados
    let filteredCentros: any[] = [];

    let mapElement: HTMLDivElement;
    let leafletMap: Map | undefined;

    let markersLayer: any | undefined;

    let userMarker: Marker | undefined;
    let customIcons: { [key: string]: Icon } = {};

    let isDarkMode: boolean = false;
    // Bandera para el modo de selección manual de ubicación
    let isAwaitingManualLocation: boolean = false;

    // SE HAN ELIMINADO: countdownTimer y timerInterval
    // SE HA ELIMINADO: clearLocationTimer()

    const filterTypes = ["Todos", "Hospital", "Clínica", "EBAIS"];
    const validCenterTypes = ["Hospital", "Clínica", "EBAIS"];

    // --- FUNCIÓN DE CÁLCULO DE DISTANCIA (Haversine) ---
    function toRad(Value: number) {
        return (Value * Math.PI) / 180;
    }

    function haversineDistance(
        lat1: number,
        lon1: number,
        lat2: number,
        lon2: number,
    ): string {
        const R = 6371; // Radio de la Tierra en kilómetros
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

        return distance.toFixed(2) + " km";
    }
    // --- FIN FUNCIONES DE DISTANCIA ---

    // --- FUNCIÓN PARA AJUSTAR EL ZOOM AUTOMÁTICO (E) ---
    function calculateAndFitBounds(data: any[]) {
        if (!leafletMap || data.length === 0) {
            return;
        }

        const L = (window as any).L;

        const allPoints: LatLngBoundsExpression = data.map((centro) => [
            centro.latitud,
            centro.longitud,
        ]);
        if (userMarker) {
            allPoints.push(userMarker.getLatLng());
        }

        if (allPoints.length === 0) return;

        const bounds = L.latLngBounds(allPoints);

        if (allPoints.length === 1) {
            leafletMap.setView(allPoints[0], 12);
        } else {
            leafletMap.fitBounds(bounds, {
                padding: [50, 50],
                maxZoom: 12,
            });
        }
    }
    // --- FIN AJUSTE DE ZOOM ---

    // FUNCIÓN PARA ALTERNAR EL MODO OSCURO (sin cambios)
    function toggleDarkMode() {
        isDarkMode = !isDarkMode;
        localStorage.setItem("darkMode", isDarkMode.toString());

        if (typeof document !== "undefined") {
            document.body.classList.toggle("dark-mode", isDarkMode);
            leafletMap?.invalidateSize();
        }
    }

    // 2. FUNCIÓN PARA OBTENER EL ÍCONO (sin cambios)
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
            iconSize: [20, 33],
            iconAnchor: [10, 33],
            popupAnchor: [1, -28],
            shadowSize: [33, 33],
        });

        customIcons[typeKey] = newIcon;
        return newIcon;
    }

    // 3. FUNCIÓN PARA DIBUJAR MARCADORES
    function addMarkersToMap(data: any[]) {
        if (!leafletMap || !markersLayer) return;

        markersLayer.clearLayers();

        const L = (window as any).L;

        const userLat = userMarker ? userMarker.getLatLng().lat : null;
        const userLng = userMarker ? userMarker.getLatLng().lng : null;
        const userCoordsParam =
            userLat !== null && userLng !== null ? `${userLat},${userLng}` : "";

        data.forEach((centro) => {
            const lat = centro.latitud;
            const lng = centro.longitud;
            const destCoords = `${lat},${lng}`;

            const icon = getCustomIcon(centro.tipo);

            let googleMapsUrl: string;
            let wazeUrl: string;
            let routingInfo = "";
            let distanceInfo = "";

            let distanceToDisplay = centro.distance_display;
            if (!distanceToDisplay && userLat !== null && userLng !== null) {
                distanceToDisplay = haversineDistance(
                    userLat,
                    userLng,
                    lat,
                    lng,
                );
            }

            if (userCoordsParam && distanceToDisplay) {
                distanceInfo = `<p style="margin: 5px 0 0; font-weight: bold;">Distancia Lineal: ${distanceToDisplay}</p>`;

                // Formato de Google Maps: ?saddr=origen&daddr=destino
                googleMapsUrl = `https://www.google.com/maps/dir/?api=1&origin=${userCoordsParam}&destination=${destCoords}&travelmode=driving`;
                wazeUrl = `https://waze.com/ul?ll=${destCoords}&navigate=yes&from_coord=${userCoordsParam}`;

                routingInfo =
                    '<p style="margin: 0; font-size: 0.8em; font-weight: bold; color: var(--secondary-color);">¡Ruta calculada desde tu ubicación!</p>';
            } else {
                distanceInfo = "";
                // Solo destino si no hay origen
                googleMapsUrl = `https://www.google.com/maps/search/?api=1&query=${destCoords}`;
                wazeUrl = `https://waze.com/ul?ll=${destCoords}&navigate=yes`;

                routingInfo =
                    '<p style="margin: 0; font-size: 0.8em; color: #888;">Para calcular la ruta automáticamente, presiona "📍 Mostrar mi Ubicación" primero.</p>';
            }

            const popupContent = `
                <div style="max-width: 250px;">
                    <h4 style="margin-bottom: 5px; color: var(--accent-color);">${centro.nombre}</h4>
                    ${routingInfo} 
                    ${distanceInfo} 
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
                .bindTooltip(centro.nombre, {
                    permanent: false,
                    direction: "top",
                })
                .addTo(markersLayer);
        });
    }

    // --- FUNCIÓN CENTRAL: Establecer la ubicación del usuario ---
    function setUserLocation(lat: number, lng: number) {
        if (!leafletMap) return;

        const L = (window as any).L;
        const latlng: LatLngExpression = [lat, lng];

        if (userMarker) {
            leafletMap.removeLayer(userMarker);
        }

        const userIcon = L.circleMarker(latlng, {
            radius: 8,
            fillColor: "#0078FF",
            color: "#000",
            weight: 1,
            opacity: 1,
            fillOpacity: 0.8,
        });

        userMarker = userIcon.addTo(leafletMap);
        userMarker
            .bindPopup(`¡Estás Aquí! (${lat.toFixed(4)}, ${lng.toFixed(4)})`)
            .openPopup();
        leafletMap.setView(latlng, 14); // Centrar mapa en la nueva ubicación

        isAwaitingManualLocation = false;
        leafletMap.off("click", onMapClick); // Desactiva el modo de clic manual

        if (centros_ccss.length > 0) {
            // Activa el re-cálculo de la distancia
            searchTerm = searchTerm.trim() + " ";
            searchTerm = searchTerm.trim();
        }
    }

    // --- FUNCIÓN DE FALLBACK: Manejar el clic en el mapa ---
    function onMapClick(e: any) {
        if (isAwaitingManualLocation && leafletMap) {
            setUserLocation(e.latlng.lat, e.latlng.lng);
            alert(
                `¡Ubicación establecida manualmente en: ${e.latlng.lat.toFixed(4)}, ${e.latlng.lng.toFixed(4)}!`,
            );
        }
    }

    // 4. FUNCIÓN PARA BUSCAR LA UBICACIÓN DEL USUARIO (SIN CONTADOR)
    function locateUser() {
        if (!leafletMap) return;

        // Alerta informativa
        alert("Buscando tu ubicación... Esto puede tardar hasta 15 segundos.");

        // Configuración de Leaflet, usa el timeout interno de 15s
        leafletMap.locate({
            setView: false,
            maxZoom: 14,
            enableHighAccuracy: true,
            timeout: 15000,
            maximumAge: 0,
        });

        // Caso de éxito: Ubicación automática encontrada
        leafletMap.once("locationfound", function (e) {
            setUserLocation(e.latlng.lat, e.latlng.lng);
        });

        // Caso de fallo: El sistema no pudo obtener una ubicación precisa
        leafletMap.once("locationerror", function (e) {
            isAwaitingManualLocation = true;
            leafletMap.on("click", onMapClick);

            // Muestra la alerta explicando el fallback de CLIC MANUAL
            alert(
                "🚨 ¡Ubicación Automática Fallida! 🚨\n\n" +
                    "Motivo: La ubicación exacta no se encontró en el tiempo límite (15s) o tu dispositivo no tiene GPS. " +
                    "Ahora puedes hacer **CLIC DIRECTAMENTE en el mapa** donde te encuentras para establecer tu ubicación manualmente.",
            );

            if (leafletMap.getZoom() < 10) {
                leafletMap.setView([9.95, -84.05], 10);
            }
        });
    }

    // --- NUEVA FUNCIÓN: Centrar el mapa en un centro de la lista ---
    function centerMapOnCenter(lat: number, lng: number, name: string) {
        if (!leafletMap) return;

        leafletMap.setView([lat, lng], 14);
    }

    // 5. LÓGICA REACTIVA DE FILTRADO, CÁLCULO DE DISTANCIA Y ORDENAMIENTO
    $: {
        if (centros_ccss.length === 0) {
            filteredCentros = [];
        } else {
            const lowerCaseSearchTerm = searchTerm.toLowerCase().trim();
            const userLat = userMarker?.getLatLng().lat;
            const userLng = userMarker?.getLatLng().lng;

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

            // 3. CÁLCULO DE DISTANCIA Y ORDENAMIENTO
            if (userLat !== undefined && userLng !== undefined) {
                data = data
                    .map((centro) => {
                        const distanceVal = haversineDistance(
                            userLat,
                            userLng,
                            centro.latitud,
                            centro.longitud,
                        );
                        const distanceNum = parseFloat(
                            distanceVal.replace(" km", ""),
                        );
                        return {
                            ...centro,
                            distance_num: distanceNum,
                            distance_display: distanceVal,
                        };
                    })
                    .sort((a, b) => a.distance_num - b.distance_num);
            } else {
                data.sort((a, b) => a.nombre.localeCompare(b.nombre));
            }

            filteredCentros = data;
        }

        // AJUSTE DE ZOOM AUTOMÁTICO
        if (leafletMap) {
            calculateAndFitBounds(filteredCentros);
        }
    }

    // BLOQUE 5b: Dibuja los marcadores
    $: if (filteredCentros && leafletMap) {
        addMarkersToMap(filteredCentros);
    }

    // 6. FUNCIÓN PARA CAMBIAR EL FILTRO DE TIPO (sin cambios)
    function setFilter(filterType: "Todos" | "Hospital" | "Clínica" | "EBAIS") {
        selectedFilter = filterType;
    }

    // 7. FUNCIÓN PARA INICIALIZAR EL MAPA
    function initializeMap(data: any[]) {
        if (!mapElement || leafletMap || !(window as any).L) return;

        const L = (window as any).L;

        leafletMap = L.map(mapElement).setView([9.95, -84.05], 8);

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

        if (typeof L.markerClusterGroup === "function") {
            markersLayer = L.markerClusterGroup();
            layerName = "Centros CCSS (CLUSTERS)";
        } else {
            markersLayer = L.layerGroup();
        }

        markersLayer.addTo(leafletMap);

        const overlayLayers = {
            [layerName]: markersLayer,
        };

        osmStandard.addTo(leafletMap);
        L.control
            .layers(baseLayers, overlayLayers, { collapsed: false })
            .addTo(leafletMap);

        // --- AGREGAR LEYENDA (Control de colores) ---
        const Legend = L.Control.extend({
            onAdd: function (map: Map) {
                const div = L.DomUtil.create("div", "info legend");
                div.innerHTML = `
                    <h4>Tipos de Centros</h4>
                    <div><span style="background-color: red;"></span> Hospital</div>
                    <div><span style="background-color: blue;"></span> Clínica</div>
                    <div><span style="background-color: green;"></span> EBAIS</div>
                `;
                L.DomEvent.disableClickPropagation(div);
                return div;
            },
        });

        new Legend({ position: "bottomright" }).addTo(leafletMap);
    }

    onMount(async () => {
        const L = await import("leaflet");
        (window as any).L = L;

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

        try {
            const response = await fetch("/datos_ccss.json");
            if (!response.ok) {
                throw new Error(
                    `Error al cargar datos: ${response.statusText}`,
                );
            }
            const data = await response.json();
            centros_ccss = data;
            filteredCentros = data;

            // FIX: Inicializar el mapa después de un pequeño retraso (0ms) para que MarkerCluster cargue
            setTimeout(() => {
                initializeMap(centros_ccss);
            }, 0);
        } catch (error) {
            console.error(
                "Ocurrió un error al procesar el JSON o inicializar el mapa.",
                error,
            );
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

        {#if isAwaitingManualLocation}
            <div class="manual-locate-warning">
                ⚠️ **Haga Clic en el Mapa** para establecer su ubicación.
            </div>
        {/if}
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

        <div class="center-list-container">
            {#if centros_ccss.length === 0}
                <p class="loading-message">Cargando datos...</p>
            {:else if filteredCentros.length > 0}
                <p class="list-summary">
                    Mostrando {filteredCentros.length} centros {userMarker
                        ? "ordenados por distancia"
                        : "ordenados alfabéticamente"}:
                </p>

                <ul class="center-list">
                    {#each filteredCentros as centro (centro.nombre + centro.latitud + centro.longitud)}
                        <li
                            class="center-list-item"
                            on:click={() =>
                                centerMapOnCenter(
                                    centro.latitud,
                                    centro.longitud,
                                    centro.nombre,
                                )}
                        >
                            <div class="center-info">
                                <h5 class="center-name">{centro.nombre}</h5>
                                <p class="center-type">
                                    {centro.tipo} ({centro.tipo_ccss})
                                </p>
                            </div>
                            {#if centro.distance_display}
                                <span class="center-distance">
                                    {centro.distance_display}
                                </span>
                            {/if}
                        </li>
                    {/each}
                </ul>
            {:else}
                <p class="loading-message">
                    No se encontraron centros con los filtros aplicados.
                </p>
            {/if}
        </div>
    </div>

    <div id="map" bind:this={mapElement}></div>
</main>

<style>
    /* ----------------------------------------------------- */
    /* VARIABLES CSS PARA MODO CLARO Y OSCURO */
    /* ----------------------------------------------------- */

    :global(body) {
        --bg-color: #ffffff;
        --text-color: #333333;
        --sidebar-bg: #f8f9fa;
        --border-color: #cccccc;
        --accent-color: #004a8b;
        --accent-color-hover: #003663;
        --secondary-color: #28a745;
        --shadow-color: rgba(0, 0, 0, 0.1);

        background-color: var(--bg-color);
        color: var(--text-color);
        transition:
            background-color 0.3s,
            color 0.3s;
        height: 100vh;
        overflow: hidden;
        font-family: "Helvetica Neue", Arial, sans-serif;
    }

    :global(body.dark-mode) {
        --bg-color: #121212;
        --text-color: #f5f5f5;
        --sidebar-bg: #1e1e1e;
        --border-color: #333333;
        --accent-color: #42a5f5;
        --accent-color-hover: #2196f3;
        --secondary-color: #5cb85c;
        --shadow-color: rgba(255, 255, 255, 0.1);
    }

    /* ----------------------------------------------------- */
    /* ESTILOS DE LAYOUT Y BARRA LATERAL */
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

    /* ESTILO CONTADOR REGRESIVO (MANTENIDO SOLO POR SI ACASO, PERO NO SE USA) */
    .countdown-timer {
        margin-top: 10px;
        padding: 8px;
        background-color: #f0ad4e;
        color: #333;
        border-radius: 4px;
        text-align: center;
        font-weight: bold;
        animation: none;
    }

    :global(body.dark-mode .countdown-timer) {
        background-color: #ff9800;
        color: #121212;
    }

    /* Advertencia de ubicación manual */
    .manual-locate-warning {
        background-color: #ffc107;
        color: #333;
        padding: 10px;
        border-radius: 4px;
        margin-bottom: 15px;
        font-weight: bold;
        text-align: center;
        box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
    }
    :global(body.dark-mode .manual-locate-warning) {
        background-color: #f57f17;
        color: #fff;
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
    /* ESTILOS DE LA LISTA LATERAL */
    /* ----------------------------------------------------- */
    .center-list-container {
        margin-top: 10px;
        padding-top: 10px;
        border-top: 1px solid var(--border-color);
    }

    .list-summary {
        font-weight: bold;
        color: var(--accent-color);
        margin-bottom: 10px;
        font-size: 0.9em;
    }

    .center-list {
        list-style: none;
        padding: 0;
        margin: 0;
    }

    .center-list-item {
        display: flex;
        justify-content: space-between;
        align-items: center;
        padding: 10px;
        margin-bottom: 8px;
        background-color: var(--bg-color);
        border: 1px solid var(--border-color);
        border-radius: 4px;
        cursor: pointer;
        transition:
            background-color 0.15s ease,
            transform 0.1s ease;
        box-shadow: 0 1px 3px var(--shadow-color);
    }

    :global(body.dark-mode .center-list-item) {
        background-color: #242424;
        border-color: #444;
    }

    .center-list-item:hover {
        background-color: var(--sidebar-bg);
        transform: translateY(-1px);
    }
    :global(body.dark-mode .center-list-item:hover) {
        background-color: #2a2a2a;
    }

    .center-info {
        flex-grow: 1;
        margin-right: 10px;
        overflow: hidden;
    }

    .center-name {
        margin: 0;
        font-size: 1em;
        font-weight: bold;
        color: var(--text-color);
        white-space: nowrap;
        overflow: hidden;
        text-overflow: ellipsis;
    }

    .center-type {
        margin: 0;
        font-size: 0.8em;
        color: #888;
        margin-top: 2px;
    }

    :global(body.dark-mode .center-type) {
        color: #bbb;
    }

    .center-distance {
        font-weight: bold;
        font-size: 0.9em;
        color: var(--secondary-color);
        flex-shrink: 0;
    }
    /* FIN ESTILOS DE LA LISTA LATERAL */

    /* Estilos globales de Leaflet */

    :global(body.dark-mode .leaflet-control-zoom a) {
        background-color: var(--sidebar-bg);
        border-bottom: 1px solid var(--border-color);
        color: var(--text-color) !important;
    }

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

    :global(body.dark-mode .leaflet-popup-content-wrapper),
    :global(body.dark-mode .leaflet-popup-tip) {
        background: var(--sidebar-bg);
        color: var(--text-color);
        box-shadow: 0 3px 14px var(--shadow-color);
    }

    :global(.leaflet-popup-content h4) {
        color: var(--accent-color) !important;
    }
    :global(body.dark-mode .leaflet-popup-content p) {
        color: var(--text-color);
    }
    :global(body.dark-mode .leaflet-popup-close-button) {
        color: var(--text-color);
    }

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

    :global(.info.legend) {
        background: var(--sidebar-bg);
        padding: 6px 8px;
        font:
            14px/16px Arial,
            Helvetica,
            sans-serif;
        box-shadow: 0 0 15px var(--shadow-color);
        border-radius: 5px;
        color: var(--text-color);
    }
    :global(.info.legend h4) {
        margin: 0 0 5px;
        color: var(--text-color);
        font-size: 1.1em;
        font-weight: bold;
    }
    :global(.info.legend div) {
        margin-bottom: 3px;
        display: flex;
        align-items: center;
    }
    :global(.info.legend span) {
        width: 15px;
        height: 15px;
        margin-right: 5px;
        border-radius: 50%;
        display: inline-block;
        border: 1px solid #333;
    }

    :global(body.dark-mode .info.legend span) {
        border: 1px solid #ccc;
    }
    :global(body.dark-mode .leaflet-tooltip) {
        background: var(--sidebar-bg);
        color: var(--text-color);
        border: 1px solid var(--border-color);
    }
</style>
