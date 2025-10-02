<script setup lang="ts">
import billy from '../assets/billy.png';
import arrow from '../assets/arrow.png';
import { computed, ref } from 'vue';

const factor = 2;
const billyHeight = `${200 * factor}px`;
const arrowDimension = `${300 * factor}px`;

// Backyard Builder coordinates
const bbLat = 37.551447;
const bbLon = 127.047016;

// User coordinates (for testing purposes, replace with actual geolocation data)
let _lat: number | undefined = undefined;
let _lon: number | undefined = undefined;

// Text values
const name = ref<string>('Backyard Builder');
const distance = ref<string>('');

//#region Error handling
const fail = (msg: string | undefined) => {
    name.value = 'Could not get your current location';
    distance.value = msg ? `${msg}. Try adding this app to your homescreen.` : 'Try adding this app to your homescreen.';
}
//#endregion

//#region Geolocation
const initGeolocation = () => {
    if (!navigator.geolocation) {
        fail(undefined);
        return;
    }

    navigator.geolocation.getCurrentPosition(
        (position) => {
            _lat = position.coords.latitude;
            _lon = position.coords.longitude;
            updateDistance();
        },
        (error) => {
            fail(error.message);
        },
        { enableHighAccuracy: true }
    );

    // Watch position for updates
    navigator.geolocation.watchPosition(
        (position) => {
            _lat = position.coords.latitude;
            _lon = position.coords.longitude;
            updateDistance();
        },
        (error) => {
            fail(error.message);
        },
        { enableHighAccuracy: true }
    );
};

const updateDistance = () => {
    if (_lat === undefined || _lon === undefined) {
        return;
    }

    const earthRadiusKm = 6371;
    
    const dLat = (bbLat - _lat) * Math.PI / 180;
    const dLon = (bbLon - _lon) * Math.PI / 180;
    
    const a = Math.sin(dLat / 2) * Math.sin(dLat / 2) +
                Math.cos(_lat * Math.PI / 180) * Math.cos(bbLat * Math.PI / 180) *
                Math.sin(dLon / 2) * Math.sin(dLon / 2);
    
    const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
    const dst = earthRadiusKm * c;
    
    distance.value = `${dst.toFixed(0)} km`;
}
//#endregion


//#region Orientation handling
const compassAvailable = ref(false);
const arrowRotation = ref(0);
const rotationRule = computed(() => `rotate(${arrowRotation.value}deg)`);

const startOrientationTracking = () => {
    // iOS 13+ requires permission to access device orientation
    if (typeof DeviceOrientationEvent !== 'undefined' && typeof (DeviceOrientationEvent as any).requestPermission === 'function') {
        (DeviceOrientationEvent as any).requestPermission()
            .then((permissionState: any) => {
                if (permissionState === 'granted') {
                    window.addEventListener('deviceorientation', updateOrientation);
                } else {
                    console.warn('Permission to access device orientation was denied.');
                }
            })
            .catch(console.error);
    } else {
        // Non-iOS devices
        window.addEventListener('deviceorientation', updateOrientation);
    }
};

const updateOrientation = (event: DeviceOrientationEvent) => {
    // If orientation data is not absolute or alpha is null, compass is not available
    if (!event.absolute || event.alpha === null) {
        compassAvailable.value = false;
        window.removeEventListener('deviceorientation', updateOrientation);
        return;
    }

    // Step 0: Ensure we have user location data
    if (_lat === undefined || _lon === undefined) {
        console.warn('User location is not available yet.');
        return;
    }
    
    // Step 0.5: Mark compass as available
    compassAvailable.value = true;

    // Step 1: Convert coordinates from degrees to radians for mathematical calculations
    const lat1Rad = _lat * Math.PI / 180;
    const lat2Rad = bbLat * Math.PI / 180;
    const deltaLonRad = (bbLon - _lon) * Math.PI / 180;
    
    // Step 2: Calculate the bearing using the forward azimuth formula
    // This gives us the angle from north to the target point
    const y = Math.sin(deltaLonRad) * Math.cos(lat2Rad);
    const x = Math.cos(lat1Rad) * Math.sin(lat2Rad) - 
              Math.sin(lat1Rad) * Math.cos(lat2Rad) * Math.cos(deltaLonRad);
    
    // Step 3: Convert the result to degrees and normalize to 0-360 range
    let bearingFromNorth = Math.atan2(y, x) * 180 / Math.PI;
    bearingFromNorth = (bearingFromNorth + 360) % 360;

    // Step 4: Calculate the relative bearing by subtracting device orientation
    // This tells us which direction to turn from where the device is currently pointing
    let relativeBearing = bearingFromNorth - event.alpha;

    // Step 5: Normalize the relative bearing to -180 to +180 range
    // Positive means turn right, negative means turn left
    if (relativeBearing > 180) relativeBearing -= 360;
    if (relativeBearing < -180) relativeBearing += 360;
    
    // Step 6: Update the arrow rotation to point towards the target
    arrowRotation.value = relativeBearing;
};

//#endregion

//#region Initialization
const init = () => {
    Promise.all([
        initGeolocation(),
        startOrientationTracking(),
    ]);
};
//#endregion
</script>

<template>
    <div id="center-content">
        <img id="icon" :src="billy" @click="init" title="Click to initialize compass">
        <img id="arrow" :class="!compassAvailable ? 'hidden' : ''" :src="arrow">
    </div>
    <span id="name">{{ name }}</span>
    <span id="distance">{{ distance }}</span>
</template>

<style scoped>
.hidden {
    visibility: hidden;
}

#center-content {
    position: relative;
    display: flex;
    justify-content: center;
    align-items: center;
    width: v-bind(arrowDimension);
    height: v-bind(arrowDimension);
}

#icon {
    z-index: 1;
    height: v-bind(billyHeight);
    position: absolute;
    transform: scale(1);
    transition: transform 0.3s ease-in-out;
}

#icon:hover {
    transform: scale(1.05);
    transition: transform 0.3s ease-in-out;
}

#arrow {
    width: v-bind(arrowDimension);
    height: v-bind(arrowDimension);
    transform-origin: center;
    z-index: -1;
    transform: v-bind(rotationRule);
}

#name {
    display: block;
    font-size: 1.5rem;
    margin-top: 1rem;
    font-weight: bold;
}

#distance {
    display: block;
    font-size: 1.2rem;
    margin-top: 0.5rem;
    color: gray;
}
</style>