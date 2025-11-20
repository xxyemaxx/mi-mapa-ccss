<script lang="ts">
    import { onMount } from "svelte";
    import type { Map, Marker, LayerGroup } from "leaflet";

    // 1. VARIABLES DE ESTADO
    let centros_ccss: any[] = [];
    let selectedFilter: "Todos" | "Hospital" | "Clínica" | "EBAIS" = "Todos";
    let searchTerm: string = ""; // ¡NUEVO! Variable para el texto de búsqueda

    let mapElement: HTMLDivElement;
    let leafletMap: Map | undefined;
    let markersLayer: LayerGroup | undefined;

    const filterTypes = ["Todos", "Hospital", "Clínica", "EBAIS"];

    // 2. FUNCIÓN PARA DIBUJAR MARCADORES
    function addMarkersToMap(data: any[]) {
        if (!leafletMap) return;

        // Elimina la capa de marcadores actual
        if (markersLayer) {
            leafletMap.removeLayer(markersLayer);
        }

        const L = (window as any).L;
        markersLayer = L.layerGroup().addTo(leafletMap);

        // Agrega solo los marcadores de la data filtrada
        data.forEach((centro) => {
            const lat = centro.latitud;
            const lng = centro.longitud;

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

            L.marker([lat, lng]).bindPopup(popupContent).addTo(markersLayer);
        });
    }

    // 3. LÓGICA REACTIVA DE FILTRADO Y BÚSQUEDA
    // Se ejecuta automáticamente CADA VEZ que cambian centros_ccss, leafletMap, selectedFilter o searchTerm
    $: if (centros_ccss.length > 0 && leafletMap) {
        const lowerCaseSearchTerm = searchTerm.toLowerCase().trim();

        // 1. Filtrar por TIPO (Hospital, Clínica, EBAIS)
        let filteredData = centros_ccss;
        if (selectedFilter !== "Todos") {
            filteredData = centros_ccss.filter(
                (centro) => centro.tipo === selectedFilter,
            );
        }

        // 2. Filtrar por BÚSQUEDA DE TEXTO (si el campo de búsqueda no está vacío)
        if (lowerCaseSearchTerm) {
            filteredData = filteredData.filter(
                (centro) =>
                    // Busca coincidencias en nombre, dirección o contacto
                    centro.nombre.toLowerCase().includes(lowerCaseSearchTerm) ||
                    centro.direccion
                        .toLowerCase()
                        .includes(lowerCaseSearchTerm) ||
                    centro.contacto.toLowerCase().includes(lowerCaseSearchTerm),
            );
        }

        // 3. Renderizar los marcadores finales
        addMarkersToMap(filteredData);
    }

    // 4. FUNCIÓN PARA CAMBIAR EL FILTRO DE TIPO (No necesita renderizar, el bloque $: lo hace)
    function setFilter(filterType: "Todos" | "Hospital" | "Clínica" | "EBAIS") {
        selectedFilter = filterType;
    }

    // 5. FUNCIÓN PARA INICIALIZAR EL MAPA
    function initializeMap(data: any[]) {
        if (!mapElement || leafletMap) return;

        const L = (window as any).L;

        // Configuración de Íconos
        const iconDefault = L.icon({
            iconUrl:
                "https://unpkg.com/leaflet@1.9.4/dist/images/marker-icon.png",
            shadowUrl:
                "https://unpkg.com/leaflet@1.9.4/dist/images/marker-shadow.png",
            iconSize: [25, 41],
            iconAnchor: [12, 41],
            popupAnchor: [1, -34],
            shadowSize: [41, 41],
        });
        L.Marker.prototype.options.icon = iconDefault;

        // Inicializar el Mapa
        leafletMap = L.map(mapElement).setView([9.95, -84.05], 8);

        // Añadir Capa de Mosaicos (Tiles) - OpenStreetMap
        L.tileLayer("https://tile.openstreetmap.org/{z}/{x}/{y}.png", {
            maxZoom: 19,
            attribution:
                '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
        }).addTo(leafletMap);

        // La primera vez, el bloque $: se encarga de renderizar los 'Todos'
    }

    onMount(async () => {
        // Importación Dinámica de Leaflet y referencia global
        const L = await import("leaflet");
        (window as any).L = L;

        // FETCH DE DATOS DESDE LA CARPETA STATIC
        try {
            const response = await fetch("/datos_ccss.json");
            if (!response.ok) {
                throw new Error(
                    `Error al cargar datos: ${response.statusText}`,
                );
            }
            const data = await response.json();
            centros_ccss = data; // Al actualizar esta variable, el bloque $: se dispara y renderiza el mapa.

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
    <title>Mapa CCSS Interactivo</title>
</svelte:head>

<main class="page-container">
    <h2>🗺️ Mapa Interactivo de Centros de la CCSS</h2>
    <p>Utilice la búsqueda y los filtros para encontrar el centro de salud.</p>

    <div class="search-box">
        <input
            type="text"
            placeholder="🔍 Buscar por nombre, dirección o teléfono..."
            bind:value={searchTerm}
        />
    </div>

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

    <div id="map" bind:this={mapElement}></div>

    {#if centros_ccss.length === 0}
        <p style="text-align: center; color: red;">
            Cargando datos o no se encontraron datos. Verifique la consola para
            errores.
        </p>
    {/if}
</main>

<style>
    /* Estilo esencial para el contenedor y el título */
    .page-container {
        padding: 20px;
        max-width: 1200px;
        margin: 0 auto;
    }

    h2 {
        color: #004a8b;
        border-bottom: 2px solid #004a8b;
        padding-bottom: 10px;
        margin-bottom: 15px;
    }

    /* Estilo esencial para el mapa */
    #map {
        height: 80vh;
        width: 100%;
        border-radius: 8px;
        box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
        margin-top: 20px;
    }

    /* Estilos para la caja de búsqueda */
    .search-box {
        margin-bottom: 15px;
    }
    .search-box input {
        width: 100%;
        padding: 12px 15px;
        font-size: 16px;
        border: 2px solid #ccc;
        border-radius: 8px;
        box-sizing: border-box; /* Asegura que el padding no cambie el ancho total */
        transition: border-color 0.2s;
    }
    .search-box input:focus {
        border-color: #004a8b;
        outline: none;
    }

    /* Estilos para los botones de filtro */
    .filter-controls {
        display: flex;
        gap: 10px;
        margin-bottom: 15px;
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

    /* Estilo para el botón activo/seleccionado */
    .filter-button.active {
        background-color: #004a8b;
        color: white;
        box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
    }

    .filter-button:hover:not(.active) {
        background-color: #e6f0f8;
    }
</style>
