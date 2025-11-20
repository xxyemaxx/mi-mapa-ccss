<script lang="ts">
    import { onMount } from "svelte";
    // Importamos los tipos de Leaflet. NO importamos la librería completa aquí
    // para evitar errores de SSR (Server-Side Rendering). La importaremos dinámicamente.
    import type { Map, Marker } from "leaflet";

    // 1. CARGA TUS DATOS DEL JSON ADJUNTO AQUÍ
    // Asegúrate de que este array contenga la totalidad de tu archivo datos_ccss json.txt
    const centros_ccss = [
        {
            nombre: "Hospital México",
            tipo: "Hospital",
            tipo_ccss: "Nacional",
            latitud: 9.951389,
            longitud: -84.115056,
            direccion: "La Uruca, San José.",
            contacto: "2242-6700 | 2242-6817",
            servicios: "Emergencias, Consulta Externa, Especialidades",
        },
        // ... (Agrega el resto de los 59 objetos de tu archivo aquí)
        {
            nombre: "Ebais Siquirres",
            tipo: "EBAIS",
            tipo_ccss: "EBAIS",
            latitud: 10.088,
            longitud: -83.509,
            direccion: "Siquirres, Limón",
            contacto: "2716-5678",
            servicios: "Atención Primaria",
        },
    ];

    let mapElement: HTMLDivElement; // Referencia al div del mapa
    let leafletMap: Map | undefined; // Referencia a la instancia del mapa de Leaflet

    onMount(async () => {
        // 2. Importación Dinámica de Leaflet (solo en el cliente)
        const L = await import("leaflet");
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

        // 3. Inicializar el Mapa
        // Coordenadas: Centro de Costa Rica (aprox.) | Zoom: 9
        leafletMap = L.map(mapElement).setView([9.95, -84.05], 9);

        // 4. Añadir Capa de Mosaicos (Tiles) - Usando OpenStreetMap
        L.tileLayer("https://tile.openstreetmap.org/{z}/{x}/{y}.png", {
            maxZoom: 19,
            attribution:
                '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
        }).addTo(leafletMap);

        // 5. Agregar Marcadores con Pop-ups
        centros_ccss.forEach((centro) => {
            const lat = centro.latitud;
            const lng = centro.longitud;

            // Contenido HTML para el Pop-up
            const popupContent = `
                <div style="max-width: 250px;">
                    <h4 style="margin-bottom: 5px;">${centro.nombre}</h4>
                    <p style="margin: 0; font-size: 0.9em;"><strong>Tipo:</strong> ${centro.tipo} (${centro.tipo_ccss})</p>
                    <p style="margin: 0; font-size: 0.9em;"><strong>Dirección:</strong> ${centro.direccion}</p>
                    <p style="margin-top: 5px; font-size: 0.9em;">
                        <strong>Contacto:</strong> <a href="tel:${centro.contacto.split("|")[0].trim()}">${centro.contacto}</a>
                    </p>
                    <p style="margin-top: 5px; font-size: 0.8em; color: #555;">Servicios: ${centro.servicios}</p>
                </div>
            `;

            // Crear el marcador y el pop-up
            L.marker([lat, lng]).addTo(leafletMap!).bindPopup(popupContent);
        });
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
    <p>
        Visualización de **Hospitales, Clínicas y EBAIS** de la Caja
        Costarricense de Seguro Social.
    </p>

    <div id="map" bind:this={mapElement}></div>
</main>

<style>
    /* Estilo esencial para que el mapa tenga un tamaño visible */
    #map {
        height: 80vh; /* Altura de la ventana */
        width: 100%;
        border-radius: 8px;
        box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
    }

    .page-container {
        padding: 20px;
        max-width: 1200px;
        margin: 0 auto;
    }

    h2 {
        color: #004a8b; /* Color de la CCSS */
        border-bottom: 2px solid #004a8b;
        padding-bottom: 10px;
        margin-bottom: 15px;
    }
</style>
