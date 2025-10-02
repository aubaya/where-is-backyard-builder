<script setup lang="ts">
import billy from '../assets/billy.png';
import arrow from '../assets/arrow.png';
import { computed, ref } from 'vue';

const debug = true;
const dump = (obj: any) => {
    if (!debug) return;
    const debugDiv = document.getElementById('debug');
    if (debugDiv) {
        debugDiv.innerText += JSON.stringify(obj, null, 2) + '\n';
    }
}

const print = (msg: string) => {
    if (!debug) return;
    const debugDiv = document.getElementById('debug');
    if (debugDiv) {
        debugDiv.innerText += msg + '\n';
    }
}

const factor = 1.0;
const billyHeight = `${200 * factor}px`;
const arrowDimension = `${300 * factor}px`;

// Initialization launched
const initialized = ref(false);

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
const initGeolocation = (): Promise<void> => {
    return new Promise((resolve, reject) => {
        if (!navigator.geolocation) {
            fail(undefined);
            reject(new Error('Geolocation not supported'));
            return;
        }

        navigator.geolocation.getCurrentPosition(
            (position) => {
                _lat = position.coords.latitude;
                _lon = position.coords.longitude;
                updateDistance();
                resolve();
                
                // Watch position for updates after initial success
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
            },
            (error) => {
                fail(error.message);
                reject(error);
            },
            { enableHighAccuracy: true }
        );
    });
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

const startOrientationTracking = (): Promise<void> => {
    return new Promise((resolve, reject) => {
        // iOS 13+ requires permission to access device orientation
        if (typeof DeviceOrientationEvent !== 'undefined' && typeof (DeviceOrientationEvent as any).requestPermission === 'function') {
            (DeviceOrientationEvent as any).requestPermission()
                .then((permissionState: any) => {
                    if (permissionState === 'granted') {
                        window.addEventListener('deviceorientation', updateOrientation);
                        resolve();
                    } else {
                        print('Permission to access device orientation was denied.');
                        reject(new Error('Permission to access device orientation was denied.'));
                    }
                })
                .catch((error: any) => {
                    print(error);
                    reject(error);
                });
        } else {
            // Non-iOS devices
            window.addEventListener('deviceorientation', updateOrientation);
            resolve();
        }
    });
};

const updateOrientation = (event: DeviceOrientationEvent) => {
    // If orientation data is not absolute or alpha is null, compass is not available
    // (unless we're on iOS, because WebKit)
    let _alpha: number | undefined = undefined;
    
    dump(event);

    if ((event as any).webkitCompassHeading) {
        _alpha = (event as any).webkitCompassHeading; // iOS
        print(`iOS detected, using webkitCompassHeading: ${_alpha}`);
    }

    if (event.absolute && event.alpha !== null && event.alpha !== undefined) {
        _alpha = event.alpha; // Non-iOS
        print(`Non-iOS detected, using alpha: ${_alpha}`);
    }

    if (_alpha === undefined) {
        print('Device orientation data is not available. No alpha');
        compassAvailable.value = false;
        window.removeEventListener('deviceorientation', updateOrientation);
        return;
    }

    // Step 0: Ensure we have user location data
    if (_lat === undefined || _lon === undefined) {
        print('No user location data available yet.');
        print('User location is not available yet.');
        return;
    }

    print(`User location: lat=${_lat}, lon=${_lon}. Ya!`);
    
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
    let relativeBearing = bearingFromNorth - _alpha;

    // Step 5: Normalize the relative bearing to -180 to +180 range
    // Positive means turn right, negative means turn left
    if (relativeBearing > 180) relativeBearing -= 360;
    if (relativeBearing < -180) relativeBearing += 360;
    
    // Step 6: Update the arrow rotation to point towards the target
    arrowRotation.value = relativeBearing;
};

//#endregion

//#region Initialization
const init = async () => {
    initialized.value = true;
    try {
        await initGeolocation();
        await startOrientationTracking();
    } catch (e) {
        print('Failed to initialize geolocation:');
        dump(e);
    }
};
//#endregion
</script>

<template>
    <div id="center-content">
        <img id="icon" :src="billy" title="Click to initialize compass">
        <img id="arrow" :class="!compassAvailable ? 'hidden' : ''" :src="arrow">
    </div>
    <span id="name">{{ name }}</span>
    <span id="distance">{{ distance }}</span>
    <a href="#" @click="init" :class="initialized ? 'hidden' : ''" style="display: block; margin-top: 1rem; color: gray;">(Start tracking)</a>
    <div id="debug" v-if="debug">
    </div>
</template>

<style scoped>
.hidden {
    visibility: hidden;
}

#debug {
    position: fixed;
    bottom: 0;
    left: 0;
    background: rgba(255, 255, 255, 0.8);
    padding: 0.5rem;
    font-size: 0.8rem;
    max-width: 100%;
    max-height: 30vh;
    overflow: scroll;
    z-index: 10;
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