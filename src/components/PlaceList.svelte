<script>
  import {
    currentLat,
    currentLong,
    searchString,
    selectedIndex,
    geoPermissionGranted,
    showModal
  } from "../store/store.js";
  import PlaceListItem from "./PlaceListItem.svelte";
  import Searchbar from "./Searchbar.svelte";
  import { calculateDistance } from "../util.js";
  import { onMount, getContext } from "svelte";

  let map, createMarker;
  let toilets;

  onMount(() => {
    fetch("data/toilets.geojson")
      .then(response => response.json())
      .then(json => {
        toilets = json.features;
      });

    //Get references to the map and createMarker functions to enable control from this component
    const { getMap, getCreateMarker } = getContext("mapContextKey");
    map = getMap();
    createMarker = getCreateMarker();

    currentMapCenter = map.getCenter();
    map.on("dragend", e => {
      currentMapCenter = map.getCenter();
    });
  });

  let filtered = [];
  let currentMapCenter;

  function receiveKeyPress(event) {
    const key = event.detail.key;

    if ($selectedIndex < filtered.length - 1 && key === "ArrowDown") {
      selectedIndex.update(selectedIndex => selectedIndex + 1);
    } else if ($selectedIndex > 0 && key === "ArrowUp") {
      selectedIndex.update(selectedIndex => selectedIndex - 1);
    } else if (filtered.length > 0 && key === "Enter") {
      currentLat.set(filtered[$selectedIndex].geometry.coordinates[1]);
      currentLong.set(filtered[$selectedIndex].geometry.coordinates[0]);
      searchString.set("");
      showModal.set(true);
    }
  }

  $: if ($searchString.length > 0) {
    selectedIndex.set(0);

    filtered = toilets.filter(
      item =>
        item.properties.name
          .toLowerCase()
          .includes($searchString.toLowerCase()) ||
        item.properties.address
          .toLowerCase()
          .includes($searchString.toLowerCase())
    );

    let referenceCenter;
    let distance;
    if ($geoPermissionGranted) {
      referenceCenter = { lat: $currentLat, lng: $currentLong };
    } else {
      referenceCenter = currentMapCenter;
    }

    filtered.forEach(filteredItem => {
      distance = calculateDistance(
        referenceCenter.lat,
        referenceCenter.lng,
        filteredItem.geometry.coordinates[1],
        filteredItem.geometry.coordinates[0],
        "K"
      );
      filteredItem.distance = distance;
    });

    filtered.sort((first, second) => first.distance - second.distance);
  } else {
    selectedIndex.set(0);
    filtered = [];
  }
</script>

<style>
  .searchList {
    max-height: 16rem;
  }

  @media screen and (min-width: 768px) {
    .searchList {
      max-height: 32rem;
    }
  }
</style>

<div class="fixed px-2 py-4 w-full lg:w-1/3 z-10">
  <Searchbar on:keyboard={e => receiveKeyPress(e)}/>
  <div
    class="searchList w-full overflow-auto mt-1 rounded-lg bg-backgroundColor shadow"
  >
    {#each filtered as place, i} <PlaceListItem placeObj={place} key={i}
    selected={$selectedIndex === i ? true : false}/> {/each}
  </div>
</div>
