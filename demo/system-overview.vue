<template>
	<div class="toolbar-container" :class="{ 'is-pinned': isToolbarPinned }">
		<!-- 顶部工具栏保持不变，简化事件绑定 -->
		<div class="toolbar-content">
			<button
				@click="startAddingDevice"
				v-show="!isAddingMode"
				:disable="isEditMode"
			>
				添加设备
			</button>

			<el-radio-group
				v-if="isAddingMode"
				v-model="defaultDeviceIcon"
				size="large"
				fill="#6cf"
			>
				<el-radio
					v-for="item in deviceTypeList"
					:key="item.code"
					:label="item.name"
					:value="item.code"
				/>
			</el-radio-group>

			<button @click="cancelOperation" v-show="isAddingMode">取消添加</button>

			<hr />

			<button
				@click="toggleEditMode"
				v-show="!isEditMode"
				:disable="isAddingMode"
			>
				编辑/移动
			</button>
			<button @click="saveEdits" v-show="isEditMode">保存更改</button>
			<button @click="cancelOperation" v-show="isEditMode">取消编辑</button>

			<el-select
				v-model="activeMapId"
				placeholder="Select"
				style="width: 240px"
			>
				<el-option
					v-for="item in mapOptions"
					:key="item.value"
					:label="item.label"
					:value="item.value"
				/>
			</el-select>
		</div>
		<div class="menu-top-right">
			<CustomButton type="primary" @click="showDrawer = true"
				>设备管理</CustomButton
			>
		</div>
	</div>

	<div id="map" ref="mapContainer"></div>

	<!-- 设备弹窗信息列表，目前只做信息筛选显示功能，避免过于复杂 -->
	<DeviceInfoDrawer v-model:visible="showDrawer" />
</template>

<script setup lang="ts">
// --- 常量定义 ---
const IMG_WIDTH = 1200;
const IMG_HEIGHT = 800;

// --- 状态管理 ---
const mapsStore = useMapsStore();
const iconStore = useDeckIcons();
const dictStore = useDictStore();

// 业务数据
const deviceTypeList = dictStore.getDictByType("deviceType");
const activeMapId = ref<string>("deck0");
const defaultDeviceIcon = ref<string>("deviceUwb");
const mapOptions = Array.from(mapsStore.maps.values()).map((map) => ({
	label: map.regionName || "未命名区域",
	value: map.mapId,
}));

// UI 状态
const showDrawer = ref(false);
const isToolbarPinned = ref(false);
const isAddingMode = ref(false);
const isEditMode = ref(false);

// --- Leaflet 实例引用 (使用 shallowRef 或普通变量，避免 Vue 深度代理 Leaflet 对象导致性能问题) ---
let mapInstance: L.Map | null = null;
let imageOverlay: L.ImageOverlay | null = null;
let editableLayerGroup: L.FeatureGroup | null = null;
// 绘制控件引用
let drawHandler: L.Draw.Marker | null = null;

// 缓存编辑前的状态，用于撤销
let markersBackupSnapshot: Map<string, L.LatLng> = new Map();

// --- 1. 地图初始化逻辑 ---
const initMap = () => {
	// 销毁旧实例防止内存泄漏
	if (mapInstance) {
		mapInstance.remove();
	}

	// 定义 bounds: [[yMin, xMin], [yMax, xMax]]
	// 注意：L.CRS.Simple 中，第一个坐标是 Y (高度/纬度)，第二个是 X (宽度/经度)
	const bounds: L.LatLngBoundsExpression = [
		[0, 0],
		[IMG_HEIGHT, IMG_WIDTH],
	];

	mapInstance = L.map("map", {
		crs: L.CRS.Simple,
		minZoom: -2,
		maxZoom: 4,
		zoomControl: false,
		attributionControl: false,
	});

	// 添加缩放控件
	L.control.zoom({ position: "bottomleft" }).addTo(mapInstance);

	// 初始化图层组
	editableLayerGroup = new L.FeatureGroup();
	mapInstance.addLayer(editableLayerGroup);

	// 绑定绘制事件
	bindMapEvents();

	// 首次加载底图
	loadBaseMap(activeMapId.value);
};

// --- 2. 底图加载逻辑 ---
const loadBaseMap = (mapId: string) => {
	if (!mapInstance) return;

	const mapData = mapsStore.maps.get(mapId);
	if (!mapData) return;

	const bounds: L.LatLngBoundsExpression = [
		[0, 0],
		[IMG_HEIGHT, IMG_WIDTH],
	];

	if (imageOverlay) {
		imageOverlay.remove();
	}

	imageOverlay = L.imageOverlay(mapData.imagePath, bounds).addTo(mapInstance);
	mapInstance.fitBounds(bounds);
};

// --- 3. 核心：渲染 Marker (数据 -> 视图) ---
// 这是一个纯粹的渲染函数，只负责把 Store 的数据画到地图上
const renderMarkers = async () => {
	if (!mapInstance || !editableLayerGroup) return;

	// 1. 清除现有图层
	editableLayerGroup.clearLayers();

	// 2. 获取当前地图的数据
	// 注意：这里建议 iconStore 内部提供一个直接获取 array 的 getter，不要在组件里做太多过滤逻辑
	const icons = iconStore.filterByMapId(activeMapId).value;

	// 3. 批量创建 Marker
	icons.forEach((device) => {
		const { x, y, type, id } = device; // 假设后端返回的数据结构包含这些

		// 重要：坐标转换。后端如果存的是 geojson [lng, lat]，在 Simple CRS 里对应 [x, y]
		// 但 Leaflet L.marker 接受的是 [lat, lng]，也就是 [y, x]
		// 必须确认后端存的是什么。这里假设后端存的是 image pixel 坐标。
		const latLng = L.latLng(y, x);

		const iconUrl = type === "deviceRFID" ? rfidIconSrc : uwbIconSrc;

		const marker = L.marker(latLng, {
			icon: createCustomIcon(iconUrl),
			draggable: isEditMode.value, // 根据当前模式决定是否可拖拽
		});

		// 绑定元数据到内部对象 (避免污染 DOM dataset，直接挂在 Leaflet options 或 feature 上)
		// @ts-ignore
		marker.feature = {
			type: "Feature",
			properties: { ...device }, // 将整个设备对象存入
		};

		// 绑定 Popup 或 Click 事件
		bindMarkerInteractions(marker);

		editableLayerGroup!.addLayer(marker);
	});
};

// 创建统一的图标配置（解决位置偏移的核心）
const createCustomIcon = (url: string) => {
	return L.icon({
		iconUrl: url,
		iconSize: [32, 32], // 图标大小
		iconAnchor: [16, 32], // ★★★ 关键：图标的“脚尖”位置。对应 [width/2, height]
		popupAnchor: [0, -32], // Popup 弹出的位置，相对于 iconAnchor
	});
};

// --- 4. 交互逻辑 ---

const bindMarkerInteractions = (marker: L.Marker) => {
	marker.on("click", (e) => {
		if (isEditMode.value || isAddingMode.value) return;
		// 正常模式下点击逻辑：比如打开详情抽屉
		const props = e.target.feature.properties;
		console.log("点击了设备:", props);
		// showDrawer.value = true; // 等等
	});

	marker.on("dragend", (e) => {
		// 只有在编辑模式下才会有这个事件，不需要额外判断
		// 更新内部状态，但不立即发请求
		const m = e.target as L.Marker;
		console.log("位置改变:", m.getLatLng());
	});
};

// 开始添加设备
const startAddingDevice = () => {
	if (!mapInstance) return;
	isAddingMode.value = true;

	const iconUrl =
		defaultDeviceIcon.value === "deviceUwb" ? uwbIconSrc : rfidIconSrc;

	// 使用 Leaflet Draw 的 Marker 工具
	drawHandler = new L.Draw.Marker(mapInstance, {
		icon: createCustomIcon(iconUrl),
	});
	drawHandler.enable();
};

// 切换编辑模式
const toggleEditMode = () => {
	if (!editableLayerGroup) return;

	isEditMode.value = true;

	// 开启所有 marker 的拖拽
	editableLayerGroup.eachLayer((layer) => {
		if (layer instanceof L.Marker) {
			layer.dragging?.enable();
			// 备份位置用于取消
			markersBackupSnapshot.set(layer.feature.properties.id, layer.getLatLng());
		}
	});
};

// 保存编辑
const saveEdits = async () => {
	const updates: any[] = [];

	editableLayerGroup!.eachLayer((layer) => {
		if (layer instanceof L.Marker) {
			const { lat, lng } = layer.getLatLng();
			const originalProps = layer.feature.properties;

			// 检查位置是否变动，或者直接全部更新
			updates.push({
				id: originalProps.id,
				x: lng, // 注意 Simple CRS: lng = x
				y: lat, // 注意 Simple CRS: lat = y
				// ...其他字段
				locationDescription: `更新于${new Date().toLocaleTimeString()}`,
			});

			layer.dragging?.disable();
		}
	});

	try {
		// 假设 api 支持批量更新，如果不支持则用 Promise.all
		// await api.batchUpdate(updates);
		// 这里模拟循环调用
		for (const item of updates) {
			await api.put("/deviceInstall/update", item);
		}

		ElMessage.success("保存成功");
		isEditMode.value = false;
		refreshData(); // 重新拉取最新数据，确保前端状态一致
	} catch (e) {
		ElMessage.error("保存失败");
	}
};

// 取消操作（添加或编辑）
const cancelOperation = () => {
	if (isAddingMode.value) {
		drawHandler?.disable();
		isAddingMode.value = false;
	}

	if (isEditMode.value) {
		// 恢复位置
		editableLayerGroup!.eachLayer((layer) => {
			if (layer instanceof L.Marker) {
				const id = layer.feature.properties.id;
				const oldPos = markersBackupSnapshot.get(id);
				if (oldPos) {
					layer.setLatLng(oldPos);
				}
				layer.dragging?.disable();
			}
		});
		markersBackupSnapshot.clear();
		isEditMode.value = false;
	}
};

// --- 5. 事件绑定 ---
const bindMapEvents = () => {
	if (!mapInstance) return;

	// 监听绘制完成 (创建新设备)
	mapInstance.on(L.Draw.Event.CREATED, async (e: any) => {
		const layer = e.layer;
		const { lat, lng } = layer.getLatLng(); // lat=y, lng=x

		const newItem = {
			mapId: activeMapId.value,
			deviceTypeCode: defaultDeviceIcon.value,
			x: lng,
			y: lat,
			locationDescription: "新建设备-" + GlobalIncrementCode.next(),
			// ... 构建完整的 DTO
		};

		try {
			await api.post("/deviceInstall/insert", newItem);
			ElMessage.success("添加成功");
			refreshData(); // 关键：不要手动 addLayer，而是刷新数据，触发 renderMarkers
		} catch (error) {
			console.error(error);
		} finally {
			isAddingMode.value = false; // 退出添加模式
			// 继续添加？如果需要连续添加，这里可以再次调用 startAddingDevice()
		}
	});
};

// --- 6. 数据同步与监听 ---

// 封装刷新数据的逻辑
const refreshData = async () => {
	await iconStore.fetchIcons(); // Store 获取数据
	// 不需要在这里调 renderMarkers，因为下面的 watch 会负责
};

// 监听 activeMapId 变化：换底图 -> 刷数据
watch(activeMapId, (newVal) => {
	loadBaseMap(newVal);
	// 这里可能不需要手动 fetch，因为 store 内部可能有 getters 依赖 mapId
	// 但为了保险，切换地图时重新渲染
	renderMarkers();
});

// 监听 Store 中的数据变化：数据变 -> 重画 Marker
// 这是一个深层监听，确保任何数据的变动都会反应在地图上
watch(
	// ：Click -> API -> Refresh Store -> Watcher triggers renderMarkers
	() => iconStore.filterByMapId(activeMapId).value, // 响应式更新不在于依赖的值，而在于依赖的链路是否完整，比如过滤依赖两个响应式对象，
	(newVal) => {
		renderMarkers();
	},
	{ deep: true }
);

// 监听图标类型切换，如果是添加模式，要更新鼠标跟随的图标
watch(defaultDeviceIcon, (newVal) => {
	if (isAddingMode.value && drawHandler) {
		drawHandler.disable();
		startAddingDevice(); // 重启绘制工具以应用新图标
	}
});

// --- 生命周期 ---
onMounted(async () => {
	await nextTick(); // “先别急，让 Vue 把界面彻底画好，让浏览器把 CSS 宽高都算好，然后再执行 L.map(...)
	initMap();
	await refreshData(); // 初始加载数据
});

onUnmounted(() => {
	if (mapInstance) {
		mapInstance.remove();
		mapInstance = null;
	}
});
// 后续拓展
// const renderMarkers = () => {
//    editableLayerGroup.clearLayers();

//    // 1. 渲染普通设备
//    const devices = ...
//    devices.forEach(d => addDeviceMarker(d));

//    // 2. 渲染摄像头 (未来)
//    const cameras = cameraStore.list;
//    cameras.forEach(c => addCameraMarker(c));

//    // 3. 渲染围栏 (未来)
//    const fences = fenceStore.list;
//    fences.forEach(f => addFencePolygon(f));
// }
</script>

<style scoped>
/* 你的样式 */
#map {
	width: 100%;
	height: 800px; /* 确保高度明确 */
	background: #f0f0f0;
}
</style>
