<template>
    <div >
        <div id="map" style="width: 50vw; height: 100vh;border-radius: 5px;"></div>
    </div>
</template>

<script setup>
import { ref, onMounted, computed, watch } from "vue";
const map = ref(null);
// const markers = ref(new Map());
// const overlays = ref(new Map());
let markers = []
let overlays = []
const polyline = ref(null);


const props = defineProps({
    newCenter: Object,
    detailList: Array,
});

// props 변경 감지
let detailPlaces = computed(() => {
    return props.detailList
})

let newCenter = computed(() => {
    return props.newCenter
})

watch(() => detailPlaces,
    (newPlace) => {
        // console.log('변동감지')
        displayMarkersAndPolyline()
    }, { deep: true }
)

watch(() => newCenter, (newc) => {
    // console.log('newCenter', newCenter.value)
    setNewCenter(newCenter.value.lat, newCenter.value.lng)
}, { deep: true },{immediate:true})


// 지도 위치 변경
const setNewCenter = (lat, lng) => {
    // console.log('이동해요')
    map.value.setCenter(new kakao.maps.LatLng(lat, lng))
}


// 여행계획들의 평균 위치를 계산하는 함수
const calculateCenter = (places) => {
    if (places.length === 0) {
        return {
            lat: newCenter.value.lat,
            lng: newCenter.value.lng
        };   
    } else {
        const totalLat = places.reduce((acc, place) => acc + place.Lat,0);
        const totalLng = places.reduce((acc, place) => acc + place.Lng, 0);
        return {
            lat: (totalLat / places.length).toFixed(7),
            lng: (totalLng / places.length).toFixed(7)
        };
    }
}


// 마커, 오버레이 초기화
const clearMarkersAndOverlays = () => {
    // console.log('clearMarkerAndOverlays 호출')

    // markers = []
    // overlays = []
    //마커 제거
    if (markers.length !== 0) {   
        markers.forEach((marker, key) => {
            // console.log(`마커 제거 전: ${key}`, marker);
            marker.setMap(null);
            // console.log(`마커 제거 후: ${key}`, marker);
        });
        markers = []
    }
    // // markers.value.clear();
    
    //오버레이 제거
    if (overlays.length !== 0) {
        overlays.forEach((overlay, key) => {
            overlay.setMap(null);
        });
        overlays = []
    }
    // // overlays.value.clear();
    
    // 폴리라인 제거
    if (polyline.value) {
        // console.log('폴리라인제거');
        polyline.value.setMap(null);
        polyline.value = null;
    }
}



// 마커, 오버레이, 폴리라인 표시
const displayMarkersAndPolyline = () => {
    // console.log('displayMarkersAndPolyLine 호출')
    // console.log('detailplaces',detailPlaces.value)

    if (!window.kakao || !window.kakao.maps) {
        console.error('카카오맵 api 호출 x');
        return;
    }

    // 초기화
    clearMarkersAndOverlays();

    let path = []
    
    // 계획 마커 만들기
    detailPlaces.value.forEach((detailPlace, index) => {
        // console.log(detailPlace)
        const position = new kakao.maps.LatLng(
            detailPlace.Lat,
            detailPlace.Lng
        )

        // 마커 이미지 
        const imageSrc = '/image/icon/icon_location_marker.png'
        const imageSize = new kakao.maps.Size(40, 40)
        const imageOption = { offset: new kakao.maps.Point(21, 35) }

        const markerImage = new kakao.maps.MarkerImage(
            imageSrc,
            imageSize,
            imageOption
        )

        const marker = new kakao.maps.Marker({
            map: map.value,
            position: position,
            image: markerImage,
            zIndex: index
        })

        // 오버레이 만들기
        const markerLabelContent = `<div class="marker-number">${index + 1}</div>`; // 여기서 index는 마커의 순서(1부터 시작)
        
        const markerLabel = new kakao.maps.CustomOverlay({
        content: markerLabelContent,
        map: map.value,
        position: position,
        // position: marker.getPosition(),
        yAnchor: 1, // 마커 이미지의 중앙 아래에 오버레이가 오도록 설정
        zIndex: index, // 순서대로 표시하기 위해 z-index 설정
        });

        // markers.value.set(marker);
        // overlays.value.set(markerLabel);
        marker.setMap(map.value)
        markerLabel.setMap(map.value)
        markers.push(marker)
        overlays.push(markerLabel)
        path.push(position)
    })
    
    // 패스 만들기
    if (path.length !== 0) {
        polyline.value = new kakao.maps.Polyline({
            path: path,
            strokeWeight: 5,
            strokeColor: "rgba(52, 152, 219, 1)",
            strokeOpacity: 1,
            strokeStyle: "solid",
        });
        polyline.value.setMap(map.value);
    }

    const pos = calculateCenter(detailPlaces.value)
    // console.log('pos', pos)
    const newC = new kakao.maps.LatLng(pos.lat, pos.lng) 
    if (pos !== undefined) {
        map.value.setCenter(newC)
    }
}


// 지도 초기화
const initMap = () => {
    const container = document.getElementById('map');
    let center = new kakao.maps.LatLng(37.5659316,126.9744791);
    const options = {
        center: center,
        level: 5
    }

    map.value = new kakao.maps.Map(container, options);
    // console.log('로드됐니?')
    
    // 마커 표시
    displayMarkersAndPolyline();
}

const loadKaKaoMapScript = () => {
    const script = document.createElement('script');
    script.src = `//dapi.kakao.com/v2/maps/sdk.js?autoload=false&appkey=${import.meta.env.VITE_KAKAO_MAP_SERVICE_KEY}&libraries=services,clusterer`;
    script.onload = () => kakao.maps.load(initMap);
    document.head.appendChild(script);
}

onMounted(() => {
    if (!window.kakao || !window.kakao.maps) {
        loadKaKaoMapScript();
    }
    else {
        initMap();
    }

})
</script>

<style scoped>
.overlay{
    background-color: white;
    border-radius: 5px;
    box-shadow: 0 0 5px rgba(0, 0, 0, 0.8);
}
.marker-number{
    color: #fff;
    background-color: blue;
    border-radius: 50%;
    font-size: 14px;
    display: inline-block;
}
</style>