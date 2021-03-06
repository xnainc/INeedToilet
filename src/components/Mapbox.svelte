<script>
  import mapboxgl from "mapbox-gl";
  import {
    currentLat,
    currentLong,
    homeLat,
    homeLong,
    searchString,
    showModal,
    geoPermissionGranted
  } from "../store/store.js";
  import { onMount, setContext } from "svelte";
  import { mapBoxKey } from "../keys.js";
  import { inBounds } from "../util.js";

  const darkStyle = "mapbox://styles/mapbox/dark-v10?optimize=true";
  const lightStyle = "mapbox://styles/mapbox/light-v10?optimize=true";

  let map;
  let userLocationMarker;
  let markers = [];
  const detailZoomLevel = 12;
  let currentMarker;

  onMount(() => {
    mapboxgl.accessToken = mapBoxKey;
    map = new mapboxgl.Map({
      container: "map",
      style: lightStyle,
      zoom: 12,
      center: [$currentLong, $currentLat],
      maxBounds: new mapboxgl.LngLatBounds(
        [103.552401, 1.166388],
        [104.031162, 1.50694]
      )
    });

    map.on("load", () => {
      map.addSource("toilets", {
        type: "geojson",
        data: "/data/toilets.geojson",
        cluster: true,
        clusterMaxZoom: detailZoomLevel,
        clusterRadius: 40
      });

      map.addLayer({
        id: "clusters",
        type: "circle",
        source: "toilets",
        filter: ["has", "point_count"],
        paint: {
          "circle-opacity": 0.8,
          "circle-color": [
            "step",
            ["get", "point_count"],
            "#b1cbe2",
            20,
            "#8da5ba",
            50,
            "#567189"
          ],
          "circle-radius": ["step", ["get", "point_count"], 20, 20, 40, 50, 60]
        }
      });

      map.addLayer({
        id: "cluster-count",
        type: "symbol",
        source: "toilets",
        filter: ["has", "point_count"],
        layout: {
          "text-field": "{point_count_abbreviated}",
          "text-font": ["Roboto Medium", "Arial Unicode MS Bold"],
          "text-size": ["step", ["get", "point_count"], 14, 20, 18, 30, 24]
        },
        paint: {
          "text-color": "#ffffff"
        }
      });

      map.addLayer({
        id: "unclustered-point",
        type: "circle",
        source: "toilets",
        filter: ["!", ["has", "point_count"]],
        paint: {
          "circle-color": [
            "step",
            ["get", "rating"],
            "#fc8181",
            2,
            "#f6ad55",
            3,
            "#f6ad55",
            4,
            "#68d391",
            5,
            "#48bb78",
            6,
            "#4fd1c5"
          ],
          "circle-opacity": 0.8,
          "circle-radius": [
            "interpolate",
            ["linear"],
            ["zoom"],
            0,
            15,
            10,
            10,
            20,
            8
          ],
          "circle-stroke-width": 1,
          "circle-stroke-color": [
            "step",
            ["get", "rating"],
            "#f51414",
            2,
            "#eb8205",
            3,
            "#eb8205",
            4,
            "#09963f",
            5,
            "#089944",
            6,
            "#0db5a6"
          ]
        }
      });

      map.on("click", "unclustered-point", e => {
        let clickedFeature = map.queryRenderedFeatures(e.point, {
          layers: ["unclustered-point"]
        })[0];

        const clickedAddress = clickedFeature.properties.address;
        fetch("/data/toilets.geojson").then(response =>
          response.json().then(geoJson => {
            geoJson.features.forEach(feature => {
              if (feature.properties.address === clickedAddress) {
                currentLat.set(feature.geometry.coordinates[1]);
                currentLong.set(feature.geometry.coordinates[0]);
                searchString.set("");
                showModal.set(true);
              }
            });
          })
        );
      });

      map.on("mouseenter", "unclustered-point", () => {
        map.getCanvas().style.cursor = "pointer";
      });
      map.on("mouseleave", "unclustered-point", () => {
        map.getCanvas().style.cursor = "";
      });

      map.on("click", "clusters", e => {
        let features = map.queryRenderedFeatures(e.point, {
          layers: ["clusters"]
        });
        let clusterId = features[0].properties.cluster_id;
        map
          .getSource("toilets")
          .getClusterExpansionZoom(clusterId, (err, zoom) => {
            if (err) return;

            map.easeTo({
              center: features[0].geometry.coordinates,
              zoom: zoom
            });
          });
      });

      map.on("mouseenter", "clusters", () => {
        map.getCanvas().style.cursor = "pointer";
      });
      map.on("mouseleave", "clusters", () => {
        map.getCanvas().style.cursor = "";
      });

      //Hide modal if clicking anywhere on map without a marker to help navigation on small screens
      map.on("click", () => {
        if ($showModal) {
          showModal.set(false);
          document.title = "SGtoilet | Toilets in Singapore";
        }
      });
    });

    //Export the reference to the map and createMarker function for use in PlaceList component
    setContext("mapContextKey", {
      getMap: () => map,
      getCreateMarker: () => createMarker
    });
  });

  function createMarker() {
    if (currentMarker !== undefined) currentMarker.remove();

    currentMarker = new mapboxgl.Marker().setLngLat([
      $currentLong,
      $currentLat
    ]);
    currentMarker.addTo(map);

    currentMarker
      .getElement()
      .firstChild.firstChild.children[1].setAttribute("fill", "#ff4d4d");

    window.history.pushState(
      {
        lat: $currentLat,
        long: $currentLong,
        modal: $showModal
      },
      null,
      "?lat=" + $currentLat + "&long=" + $currentLong
    );
  }

  function createLocationDot() {
    if (userLocationMarker !== undefined) userLocationMarker.remove();

    let el = document.createElement("div");
    //Use existing Mapbox css and style for pulsing blue location icon
    el.className =
      "mapboxgl-user-location-dot mapboxgl-marker mapboxgl-marker-anchor-center";
    el.style = "transform: translate(-50%, -50%) translate(206px, 366px);";
    userLocationMarker = new mapboxgl.Marker(el).setLngLat([
      $currentLong,
      $currentLat
    ]);
    userLocationMarker.addTo(map);
  }

  $: if (map !== undefined) {
    map.easeTo({
      center: [$currentLong, $currentLat],
      zoom: detailZoomLevel + 1
    });

    if (
      $currentLat !== 1.29027 &&
      $currentLong !== 103.851959 &&
      $currentLat !== $homeLat &&
      $currentLong !== $homeLong
    )
      createMarker();

    if (
      $currentLat === $homeLat &&
      $currentLong === $homeLong &&
      $geoPermissionGranted
    )
      createLocationDot();
  }
</script>

<svelte:head>
  <link
    href="https://api.mapbox.com/mapbox-gl-js/v1.0.0/mapbox-gl.css"
    rel="stylesheet"
  />
</svelte:head>

<div id="map" class="w-screen h-screen"></div>
