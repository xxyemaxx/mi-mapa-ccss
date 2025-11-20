<script lang="ts">
    import { onMount } from "svelte";
    import type { Map, Marker, LayerGroup, Icon } from "leaflet";

    // 1. VARIABLES DE ESTADO
    let centros_ccss: any[] = [];
    let selectedFilter: "Todos" | "Hospital" | "Clínica" | "EBAIS" = "Todos";
    let searchTerm: string = "";

    // Lista de centros actualmente visibles (resultado del filtro)
    let visibleCentros: any[] = [];
    // ¡NUEVO! Variable para controlar si la lista está visible
    let isListOpen: boolean = false; // Iniciamos en falso para que el mapa ocupe el 100%

    let mapElement: HTMLDivElement;
    let leafletMap: Map | undefined;
    let markersLayer: LayerGroup | undefined;
    let userMarker: Marker | undefined;

    let customIcons: { [key: string]: Icon } = {};
    const filterTypes = ["Todos", "Hospital", "Clínica", "EBAIS"];

    // 2. FUNCIÓN PARA OBTENER LOS ÍCONOS PERSONALIZADOS
    function getCustomIcons(L: any) {
        // HOSPITAL (Rojo)
        const hospitalIcon = new L.Icon({
            iconUrl:
                "https://raw.githubusercontent.com/pointhi/leaflet-color-markers/master/img/marker-icon-red.png",
            shadowUrl:
                "https://cdnjs.cloudflare.com/ajax/libs/leaflet/0.7.7/images/marker-shadow.png",
            iconSize: [25, 41],
            iconAnchor: [12, 41],
            popupAnchor: [1, -34],
            shadowSize: [41, 41],
        });

        // CLÍNICA (Azul CCSS)
        const clinicaIcon = new L.Icon({
            iconUrl:
                "https://raw.githubusercontent.com/pointhi/leaflet-color-markers/master/img/marker-icon-blue.png",
            shadowUrl:
                "https://cdnjs.cloudflare.com/ajax/libs/leaflet/0.7.7/images/marker-shadow.png",
            iconSize: [25, 41],
            iconAnchor: [12, 41],
            popupAnchor: [1, -34],
            shadowSize: [41, 41],
        });

        // EBAIS (Verde)
        const ebaisIcon = new L.Icon({
            iconUrl:
                "https://raw.githubusercontent.com/pointhi/leaflet-color-markers/master/img/marker-icon-green.png",
            shadowUrl:
                "https://cdnjs.cloudflare.com/ajax/libs/leaflet/0.7.7/images/marker-shadow.png",
            iconSize: [25, 41],
            iconAnchor: [12, 41],
            popupAnchor: [1, -34],
            shadowSize: [41, 41],
        });

        // ÍCONO DE UBICACIÓN DEL USUARIO (Naranja)
        const userLocationIcon = new L.Icon({
            iconUrl:
                "https://raw.githubusercontent.com/pointhi/leaflet-color-markers/master/img/marker-icon-orange.png",
            shadowUrl:
                "https://cdnjs.cloudflare.com/ajax/libs/leaflet/0.7.7/images/marker-shadow.png",
            iconSize: [25, 41],
            iconAnchor: [12, 41],
            popupAnchor: [1, -34],
            shadowSize: [41, 41],
        });

        customIcons = {
            Hospital: hospitalIcon,
            Clínica: clinicaIcon,
            EBAIS: ebaisIcon,
            UserLocation: userLocationIcon,
        };
    }

    // 3. FUNCIÓN PARA DIBUJAR MARCADORES
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

            const popupContent = `
                <div style="max-width: 250px;">
                    <h4 style="margin-bottom: 5px; color: #004a8b;">${centro.nombre}</h4>
                    <p style="margin: 0; font-size: 0.9em;"><strong>Tipo:</strong> ${centro.tipo} (${centro.tipo_ccss})</p>
                    <p style="margin: 0; font-size: 0.9em;"><strong>Dirección:</strong> ${centro.direccion}</p>
                    <p style="margin-top: 5px; font-size: 0.9em;">
                        <strong>Contacto:</strong> <a href="tel:${centro.contacto.split("|")[0].trim()}">${centro.contacto}</a>
                    </p>
                    <p style="margin-top: 5px; font-size: 0.8em; color: #555;">Servicios: ${centro.servicios}</p>
                </div>
            `;

            const marker = L.marker([lat, lng], { icon: iconToUse })
                .bindPopup(popupContent)
                .addTo(markersLayer);

            centro.markerInstance = marker;
        });
    }

    // 4. LÓGICA REACTIVA DE FILTRADO Y BÚSQUEDA
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

        // Guardamos el resultado del filtro para la lista
        visibleCentros = filteredData;

        // Renderizar los marcadores finales
        addMarkersToMap(visibleCentros);
    }

    // 5. FUNCIÓN PARA CENTRAR EL MAPA DESDE LA LISTA
    function zoomToCenter(centro: any) {
        if (!leafletMap || !centro.markerInstance) return;

        leafletMap.setView([centro.latitud, centro.longitud], 15);

        centro.markerInstance.openPopup();
    }

    // 6. FUNCIÓN PARA UBICAR AL USUARIO (Geolocalización)
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
            const orangeIcon = customIcons["UserLocation"];

            userMarker = L.marker(e.latlng, { icon: orangeIcon })
                .addTo(leafletMap)
                .bindPopup("¡Tu Ubicación Actual!")
                .openPopup();
        });

        leafletMap.once("locationerror", function (e) {
            alert(
                "No fue posible obtener su ubicación. Asegúrese de que la geolocalización esté activada y que el permiso haya sido otorgado. (Error: " +
                    e.message +
                    ")",
            );
        });
    }

    // 7. FUNCIÓN PARA INICIALIZAR EL MAPA
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

        leafletMap = L.map(mapElement).setView([9.95, -84.05], 8);

        L.tileLayer("https://tile.openstreetmap.org/{z}/{x}/{y}.png", {
            maxZoom: 19,
            attribution:
                '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
        }).addTo(leafletMap);
    }

    onMount(async () => {
        const L = await import("leaflet");
        (window as any).L = L;

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
                class="filter-button location-button"
                on:click={findMe}
                title="Centrar mapa en su ubicación actual"
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
                            <p class="center-address">📍 {centro.direccion}</p>
                            <p class="center-contact">
                                📞 {centro.contacto.split("|")[0].trim()}
                            </p>
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
    /* 1. AMPLÍA EL CONTENEDOR PRINCIPAL (.page-container) */
    .page-container {
        padding: 10px;
        max-width: 100%;
        margin: 0 auto;
    }

    h2 {
        margin-top: 0;
        margin-bottom: 5px;
        font-size: 1.8em;
        color: #004a8b;
        border-bottom: 2px solid #004a8b;
        padding-bottom: 10px;
    }

    .controls-wrapper {
        display: flex;
        flex-direction: column;
        gap: 10px;
        margin-bottom: 15px;
    }

    /* Contenedor Flexbox para Mapa y Lista */
    .map-and-list-container {
        display: flex;
        gap: 20px;
        margin-top: 5px;
    }

    /* 2. AUMENTAR LA ALTURA DEL MAPA (#map) Y CONTROL DE ANCHO */
    #map {
        flex-grow: 1;
        height: 95vh;
        width: 70%; /* Ancho cuando la lista está abierta */
        border-radius: 8px;
        box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
        margin-top: 5px;
        transition: width 0.3s ease; /* Transición suave para el efecto de expansión */
    }

    /* Estilo cuando la lista está CERRADA: el mapa toma el 100% */
    .map-and-list-container.list-closed #map {
        width: 100%;
    }

    /* Estilos de la caja de búsqueda */
    .search-box {
        margin-bottom: 5px;
    }
    .search-box input {
        width: 100%;
        padding: 12px 15px;
        font-size: 16px;
        border: 2px solid #ccc;
        border-radius: 8px;
        box-sizing: border-box;
        transition: border-color 0.2s;
    }
    .search-box input:focus {
        border-color: #004a8b;
        outline: none;
    }

    /* Estilos de los botones de filtro */
    .filter-controls {
        display: flex;
        gap: 10px;
        flex-wrap: wrap;
    }

    .filter-button {
        padding: 10px 15px;
        border: 2px solid #004a8b;
        border-radius: 5px;
        background-color: white;
        color: #004a8b;
        font-weight: bold;
        cursor: pointer;
        transition: all 0.2s ease;
    }

    /* Estilo del botón de ubicación */
    .filter-button.location-button {
        border-color: #f7931e;
        color: #f7931e;
        order: -1;
    }

    .filter-button.location-button:hover {
        background-color: #f7931e;
        color: white;
    }

    /* Estilo del botón de alternancia de lista */
    .filter-button.list-toggle-button {
        background-color: #f0f0f0;
        color: #333;
        border-color: #aaa;
    }
    .filter-button.list-toggle-button:hover {
        background-color: #ccc;
        border-color: #999;
    }

    .filter-button.active {
        background-color: #004a8b;
        color: white;
        box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
    }

    .filter-button:hover:not(.active):not(.location-button):not(
            .list-toggle-button
        ) {
        background-color: #e6f0f8;
    }

    /* 3. Estilos de la Lista de Resultados (ancho fijo) */
    .results-list {
        width: 300px; /* Ancho fijo para la lista */
        max-height: 95vh;
        min-width: 300px;
        overflow-y: auto;
        border: 1px solid #ccc;
        border-radius: 8px;
        box-shadow: 0 4px 12px rgba(0, 0, 0, 0.05);
        padding: 10px;
        background-color: #f9f9f9;
    }

    .list-header h3 {
        margin: 0;
        padding-bottom: 10px;
        border-bottom: 1px solid #ddd;
        color: #004a8b;
    }

    .center-item {
        padding: 10px 0;
        border-bottom: 1px dashed #eee;
        cursor: pointer;
        transition: background-color 0.1s;
    }

    .center-item:hover {
        background-color: #e6f0f8;
    }

    .center-name {
        font-weight: bold;
        margin: 0 0 4px 0;
        display: flex;
        justify-content: space-between;
        align-items: center;
    }

    .center-type {
        font-size: 0.75em;
        font-weight: normal;
        padding: 3px 6px;
        border-radius: 4px;
        color: white;
    }

    /* Colores del tag según el tipo de centro */
    .center-type.hospital {
        background-color: #d9534f;
    }
    .center-type.clinica {
        background-color: #004a8b;
    }
    .center-type.ebais {
        background-color: #5cb85c;
    }

    .center-address,
    .center-contact {
        font-size: 0.85em;
        color: #666;
        margin: 0;
    }

    .no-results {
        text-align: center;
        color: #999;
        margin-top: 20px;
    }

    /* Media query para dispositivos pequeños (comportamiento de columna) */
    @media (max-width: 1000px) {
        .map-and-list-container {
            flex-direction: column;
        }
        #map {
            width: 100%;
            height: 60vh;
        }

        /* Ajuste para que el mapa ocupe 100% en móvil, sin importar la lista */
        .map-and-list-container.list-closed #map {
            width: 100%;
        }

        .results-list {
            width: 100%;
            max-height: 40vh;
        }
    }
</style>
