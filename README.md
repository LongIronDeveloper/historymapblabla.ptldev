# historymapblabla.ptldev
<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Biên Tập Bản Đồ Khởi Nghĩa Lam Sơn</title>
    
    <link href="https://unpkg.com/maplibre-gl@3.6.2/dist/maplibre-gl.css" rel="stylesheet" />
    <script src="https://unpkg.com/maplibre-gl@3.6.2/dist/maplibre-gl.js"></script>
    <script src="https://unpkg.com/@turf/turf/turf.min.js"></script>
    
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css" />
    
    <link href="https://fonts.googleapis.com/css2?family=Merriweather:ital,wght@0,400;0,700;1,400&family=Roboto:wght@400;700&display=swap" rel="stylesheet">

    <style>
        body { margin: 0; padding: 0; font-family: 'Segoe UI', sans-serif; overflow: hidden; display: flex; }
        
        /* --- SIDEBAR --- */
        #sidebar {
            width: 340px;
            height: 100vh;
            background: #2c3e50;
            color: white;
            padding: 15px;
            box-sizing: border-box;
            transition: transform 0.3s ease;
            position: absolute;
            z-index: 1000;
            left: 0;
            overflow-y: auto;
            box-shadow: 2px 0 10px rgba(0,0,0,0.5);
        }
        #sidebar.collapsed { transform: translateX(-350px); }

        /* Nút Toggle Sidebar (Góc trên phải, hình vuông nhỏ) */
        #toggle-btn {
            position: absolute;
            top: 15px;
            right: 15px; /* Góc trên phải */
            width: 35px;
            height: 35px;
            z-index: 2000;
            background: #e74c3c;
            color: white;
            border: none;
            cursor: pointer;
            border-radius: 4px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.3);
            display: flex;
            align-items: center;
            justify-content: center;
        }
        #toggle-btn:hover { background: #c0392b; }

        /* Nút Ẩn/Hiện Tên Địa Danh */
        #label-toggle-btn {
            position: absolute;
            top: 60px;
            right: 15px;
            width: 35px;
            height: 35px;
            z-index: 2000;
            background: #34495e;
            color: white;
            border: none;
            cursor: pointer;
            border-radius: 4px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.3);
            display: flex; align-items: center; justify-content: center;
        }

        h3 { margin-top: 0; border-bottom: 1px solid #555; padding-bottom: 5px; font-size: 16px; color: #f1c40f; }
        .control-group { margin-bottom: 15px; background: rgba(0,0,0,0.2); padding: 10px; border-radius: 8px; }
        .control-group label { display: block; margin-bottom: 5px; font-size: 13px; color: #ccc; }
        
        input, select, button { width: 100%; margin-bottom: 8px; padding: 6px; box-sizing: border-box; border-radius: 4px; border: 1px solid #555; background: #34495e; color: white; }
        .row { display: flex; gap: 5px; }
        .row > * { flex: 1; }

        button.btn-action { background: #27ae60; font-weight: bold; border: none; padding: 10px; }
        button.btn-action:hover { background: #2ecc71; }
        
        /* Icon Selector Grid */
        .icon-grid { display: grid; grid-template-columns: repeat(4, 1fr); gap: 5px; margin-bottom: 10px; }
        .icon-option { 
            background: #34495e; border: 1px solid #555; padding: 8px; text-align: center; cursor: pointer; border-radius: 4px; 
        }
        .icon-option:hover { background: #455a64; }
        .icon-option.selected { background: #f1c40f; color: #2c3e50; border-color: #f1c40f; }

        /* --- MAP & OVERLAYS --- */
        #map { width: 100vw; height: 100vh; background: #eee; }
        
        /* Legend */
        #legend-overlay {
            position: absolute; bottom: 20px; right: 20px;
            background: rgba(255,255,255,0.95); padding: 10px;
            border: 2px solid #333; border-radius: 5px; min-width: 180px; z-index: 900;
            display: none; color: #333; font-size: 13px;
        }

        /* Marker & Text Styles */
        .custom-text-marker { text-align: center; white-space: nowrap; pointer-events: auto; cursor: move; }
        .marker-selected { outline: 2px dashed #f1c40f; } /* Hiệu ứng khi chọn mũi tên để xoay */

    </style>
</head>
<body>

    <button id="toggle-btn" onclick="toggleSidebar()" title="Đóng/Mở Công Cụ"><i class="fas fa-cog"></i></button>
    <button id="label-toggle-btn" onclick="toggleMapLabels()" title="Ẩn/Hiện Tên Địa Danh"><i class="fas fa-font"></i></button>

    <div id="sidebar">
        <h2 style="text-align:center; color:#ecf0f1; margin: 0 0 15px 0;">Bản Đồ Chiến Thuật</h2>

        <div class="control-group">
            <h3>1. Chế Độ Vẽ</h3>
            <select id="draw-mode" onchange="updateUI()">
                <option value="none">-- Chỉ Xem / Chọn --</option>
                <option value="arrow">Mũi Tên Tiến Công (Đường)</option>
                <option value="marker">Biểu Tượng (Icon)</option>
                <option value="text">Văn Bản (Chữ)</option>
                <option value="polygon">Vùng Hoạt Động (Tô màu)</option>
            </select>
            <button class="btn-action" id="action-btn" onclick="toggleDrawing()">Bắt đầu vẽ</button>
        </div>

        <div id="options-panel">
            <div class="control-group">
                <label>Màu Sắc / Độ Mờ:</label>
                <div class="row">
                    <input type="color" id="draw-color" value="#e74c3c">
                    <input type="range" id="draw-opacity" min="0.1" max="1" step="0.1" value="1" title="Độ đậm nhạt">
                </div>
                <label>Kích thước / Độ dày:</label>
                <input type="number" id="draw-size" value="4" min="1" max="50">
                <input type="text" id="draw-label" placeholder="Nhập chú thích (Legend)...">
            </div>

            <div class="control-group" id="icon-options" style="display:none;">
                <label>Chọn Icon:</label>
                <div class="icon-grid">
                    <div class="icon-option selected" onclick="selectIcon('fire')" title="Nơi khởi nghĩa"><i class="fas fa-fire"></i></div>
                    <div class="icon-option" onclick="selectIcon('star')" title="Chiến thắng"><i class="fas fa-star"></i></div>
                    <div class="icon-option" onclick="selectIcon('chess-rook')" title="Thành trì"><i class="fas fa-chess-rook"></i></div>
                    <div class="icon-option" onclick="selectIcon('flag')" title="Sở chỉ huy"><i class="fas fa-flag"></i></div>
                    <div class="icon-option" onclick="selectIcon('skull-crossbones')" title="Bị tiêu diệt"><i class="fas fa-skull-crossbones"></i></div>
                    <div class="icon-option" onclick="selectIcon('campground')" title="Doanh trại"><i class="fas fa-campground"></i></div>
                    <div class="icon-option" onclick="selectIcon('exclamation-triangle')" title="Nguy hiểm"><i class="fas fa-exclamation-triangle"></i></div>
                    <div class="icon-option" onclick="selectIcon('circle')" title="Chấm tròn"><i class="fas fa-circle"></i></div>
                </div>
            </div>

            <div class="control-group" id="text-options" style="display:none;">
                <label>Nội dung chữ:</label>
                <input type="text" id="text-content" placeholder="Nhập nội dung hiển thị...">
                <label>Font chữ:</label>
                <select id="text-font">
                    <option value="'Roboto', sans-serif">Roboto (Hiện đại)</option>
                    <option value="'Merriweather', serif">Merriweather (Cổ điển)</option>
                </select>
                <div class="row">
                    <label><input type="checkbox" id="text-bold"> <b>Đậm</b></label>
                    <label><input type="checkbox" id="text-italic"> <i>Nghiêng</i></label>
                </div>
            </div>
        </div>

        <div class="control-group">
            <h3>Quản Lý Dữ Liệu</h3>
            <button class="btn-action" style="background:#2980b9;" onclick="exportJSON()"><i class="fas fa-download"></i> Xuất JSON</button>
            <button class="btn-action" style="background:#e67e22;" onclick="document.getElementById('file-input').click()"><i class="fas fa-upload"></i> Nhập JSON</button>
            <input type="file" id="file-input" style="display: none;" onchange="importJSON(this)" accept=".json">
            <button class="btn-action" style="background:#c0392b; margin-top:5px;" onclick="location.reload()">Xóa Hết</button>
        </div>
        
        <div style="font-size:12px; color:#aaa; font-style:italic;">
            * Mẹo: Click vào đầu mũi tên rồi nhấn A hoặc D để xoay hướng.<br>
            * Mẹo: Click vào đường mũi tên để bẻ cong.
        </div>
    </div>

    <div id="map"></div>
    <div id="legend-overlay"><h4>Chú Giải</h4><div id="legend-content"></div></div>

    <script>
        // --- 1. KHỞI TẠO BẢN ĐỒ ---
        const map = new maplibregl.Map({
            container: 'map',
            style: 'https://basemaps.cartocdn.com/gl/voyager-gl-style/style.json',
            center: [105.5, 20.0], // Vị trí Thanh Hóa/Nghệ An
            zoom: 7,
            pitch: 0,
            antialias: true
        });

        // Biến toàn cục
        let mapData = { features: [], legend: [] };
        let isDrawing = false;
        let currentDrawType = 'none';
        let currentPoints = []; // Cho đường/polygon
        let tempId = 'temp-layer';
        let selectedIcon = 'fire';
        let selectedArrowHead = null; // Marker đầu mũi tên đang được chọn để xoay

        // --- 2. XỬ LÝ BIÊN GIỚI & LABEL ---
        map.on('load', async () => {
            // Tải viền Việt Nam (Sử dụng nguồn GeoJSON mở)
            // Ở đây dùng mask bao quanh để làm nổi bật hoặc vẽ line đậm
            try {
                // Thêm Source viền VN (Link mẫu github raw)
                map.addSource('vietnam-border', {
                    type: 'geojson',
                    data: 'https://raw.githubusercontent.com/vietnam-geospatial/vietnam-administrative-boundaries/master/vietnam_boundary.geojson' 
                });
                
                // Vẽ đường viền đậm
                map.addLayer({
                    id: 'vn-border-line',
                    type: 'line',
                    source: 'vietnam-border',
                    paint: {
                        'line-color': '#2c3e50',
                        'line-width': 3,
                        'line-opacity': 0.8
                    }
                });
            } catch (e) { console.log("Không tải được border"); }
        });

        function toggleMapLabels() {
            const layers = map.getStyle().layers;
            const isVisible = layers.some(l => (l.id.includes('label') || l.id.includes('text')) && map.getLayoutProperty(l.id, 'visibility') === 'visible');
            const newValue = isVisible ? 'none' : 'visible';
            layers.forEach(layer => {
                if(layer.id.includes('label') || layer.id.includes('place') || layer.id.includes('text')) {
                    map.setLayoutProperty(layer.id, 'visibility', newValue);
                }
            });
        }

        // --- 3. LOGIC VẼ (CORE) ---
        function toggleDrawing() {
            isDrawing = !isDrawing;
            const btn = document.getElementById('action-btn');
            if(isDrawing) {
                btn.innerText = "DỪNG VẼ (ESC)";
                btn.style.background = "#c0392b";
                map.getCanvas().style.cursor = 'crosshair';
                currentDrawType = document.getElementById('draw-mode').value;
            } else {
                stopDrawing();
            }
        }

        function stopDrawing() {
            isDrawing = false;
            currentPoints = [];
            const btn = document.getElementById('action-btn');
            btn.innerText = "Bắt đầu vẽ";
            btn.style.background = "#27ae60";
            map.getCanvas().style.cursor = '';
            // Xóa temp
            if(map.getLayer(tempId)) map.removeLayer(tempId);
            if(map.getSource(tempId)) map.removeSource(tempId);
        }

        map.on('click', (e) => {
            if (!isDrawing) return;
            const color = document.getElementById('draw-color').value;
            const size = document.getElementById('draw-size').value;
            const label = document.getElementById('draw-label').value;

            // A. ICON
            if (currentDrawType === 'marker') {
                addIconMarker(e.lngLat, selectedIcon, color, size, label);
                saveFeature('marker', {lng: e.lngLat.lng, lat: e.lngLat.lat, icon: selectedIcon}, {color, size}, label);
                if(label) addToLegend('marker', color, label, selectedIcon);
            }
            
            // B. TEXT
            else if (currentDrawType === 'text') {
                const text = document.getElementById('text-content').value || "Văn bản mẫu";
                const font = document.getElementById('text-font').value;
                const isBold = document.getElementById('text-bold').checked;
                const isItalic = document.getElementById('text-italic').checked;
                
                addTextMarker(e.lngLat, text, color, size, font, isBold, isItalic);
                saveFeature('text', {lng: e.lngLat.lng, lat: e.lngLat.lat, text}, {color, size, font, isBold, isItalic}, "");
            }

            // C. ARROW (Line)
            else if (currentDrawType === 'arrow') {
                currentPoints.push([e.lngLat.lng, e.lngLat.lat]);
                updateTempLayer('line');
            }

            // D. POLYGON
            else if (currentDrawType === 'polygon') {
                currentPoints.push([e.lngLat.lng, e.lngLat.lat]);
                updateTempLayer('fill');
            }
        });

        // KẾT THÚC VẼ (Double Click)
        map.on('dblclick', (e) => {
            if(!isDrawing) return;
            e.preventDefault(); // Chặn zoom

            const color = document.getElementById('draw-color').value;
            const size = document.getElementById('draw-size').value;
            const label = document.getElementById('draw-label').value;
            const opacity = document.getElementById('draw-opacity').value;

            if (currentDrawType === 'arrow' && currentPoints.length > 1) {
                addArrowToMap(currentPoints, color, size, label);
                saveFeature('arrow', currentPoints, {color, size}, label);
                if(label) addToLegend('arrow', color, label);
                stopDrawing();
            }
            else if (currentDrawType === 'polygon' && currentPoints.length > 2) {
                // Đóng vòng polygon
                currentPoints.push(currentPoints[0]);
                addPolygonToMap(currentPoints, color, opacity, label);
                saveFeature('polygon', currentPoints, {color, opacity}, label);
                if(label) addToLegend('polygon', color, label);
                stopDrawing();
            }
        });

        // VẼ NHÁP
        function updateTempLayer(type) {
            const data = {
                type: 'Feature',
                geometry: { 
                    type: type === 'line' ? 'LineString' : 'Polygon', 
                    coordinates: type === 'line' ? currentPoints : [currentPoints] 
                }
            };

            if (map.getSource(tempId)) {
                map.getSource(tempId).setData(data);
            } else {
                map.addSource(tempId, { type: 'geojson', data: data });
                if (type === 'line') {
                    map.addLayer({ id: tempId, type: 'line', source: tempId, paint: {'line-color': '#333', 'line-dasharray': [2, 2]} });
                } else {
                    map.addLayer({ id: tempId, type: 'fill', source: tempId, paint: {'fill-color': '#333', 'fill-opacity': 0.3} });
                }
            }
        }

        // --- 4. CÁC HÀM ADD FEATURE & INTERACTION ---

        // ICON
        function selectIcon(iconName) {
            selectedIcon = iconName;
            document.querySelectorAll('.icon-option').forEach(el => el.classList.remove('selected'));
            event.currentTarget.classList.add('selected');
        }

        function addIconMarker(lngLat, icon, color, size, label) {
            const el = document.createElement('div');
            // Font size = size * 5 (vd size 4 -> 20px)
            const px = parseInt(size) * 8; 
            el.innerHTML = `<i class="fas fa-${icon}" style="color:${color}; font-size:${px}px; text-shadow: 2px 2px 2px white;"></i>`;
            new maplibregl.Marker({ element: el }).setLngLat(lngLat).addTo(map);
        }

        // TEXT
        function addTextMarker(lngLat, text, color, size, font, bold, italic) {
            const el = document.createElement('div');
            el.className = 'custom-text-marker';
            el.style.color = color;
            el.style.fontSize = (parseInt(size) * 5) + 'px';
            el.style.fontFamily = font;
            el.style.fontWeight = bold ? 'bold' : 'normal';
            el.style.fontStyle = italic ? 'italic' : 'normal';
            el.innerText = text;
            
            new maplibregl.Marker({ element: el }).setLngLat(lngLat).addTo(map);
        }

        // POLYGON
        function addPolygonToMap(coords, color, opacity, label) {
            const id = 'poly-' + Date.now();
            map.addSource(id, { type: 'geojson', data: { type: 'Feature', geometry: { type: 'Polygon', coordinates: [coords] } } });
            map.addLayer({
                id: id, type: 'fill', source: id,
                paint: { 'fill-color': color, 'fill-opacity': parseFloat(opacity) }
            });
        }

        // ARROW (Phức tạp nhất: Có mũi tên, bẻ cong, xoay)
        function addArrowToMap(coords, color, size, label) {
            const id = 'line-' + Date.now();
            
            // 1. Line
            map.addSource(id, { type: 'geojson', data: { type: 'Feature', geometry: { type: 'LineString', coordinates: coords } } });
            
            // Layer hiển thị đường
            map.addLayer({
                id: id, type: 'line', source: id,
                layout: { 'line-join': 'round', 'line-cap': 'round' },
                paint: { 'line-color': color, 'line-width': parseInt(size) }
            });

            // 2. Click vào đường để Bẻ cong
            map.on('click', id, (e) => {
                if(isDrawing) return;
                new maplibregl.Popup()
                    .setLngLat(e.lngLat)
                    .setHTML(`<button onclick="curveLine('${id}', '${color}')" style="background:#2980b9;color:white;border:none;padding:5px;">Bẻ cong hướng tiến công</button>`)
                    .addTo(map);
            });
            // Thay đổi con trỏ chuột khi hover đường
            map.on('mouseenter', id, () => map.getCanvas().style.cursor = 'pointer');
            map.on('mouseleave', id, () => map.getCanvas().style.cursor = '');

            // 3. Arrow Head (Đầu mũi tên)
            addArrowHead(coords, color, size);
        }

        function addArrowHead(coords, color, size) {
            const last = coords[coords.length - 1];
            const prev = coords[coords.length - 2];
            let bearing = turf.bearing(turf.point(prev), turf.point(last)); // Tính góc

            const headSize = parseInt(size) * 6;
            const el = document.createElement('div');
            el.innerHTML = `<i class="fas fa-location-arrow" style="transform: rotate(${bearing - 45}deg); color: ${color}; font-size: ${headSize}px; display:block; text-shadow:0 0 2px white;"></i>`;
            el.style.cursor = 'pointer';

            const marker = new maplibregl.Marker({ element: el })
                .setLngLat(last)
                .addTo(map);

            // LOGIC XOAY ĐẦU MŨI TÊN
            el.addEventListener('click', (e) => {
                e.stopPropagation(); // Không kích hoạt click map
                // Bỏ chọn cũ
                if(selectedArrowHead) selectedArrowHead.element.style.outline = 'none';
                
                // Chọn mới
                selectedArrowHead = { marker: marker, element: el, bearing: bearing };
                el.style.borderRadius = "50%";
                el.style.outline = "3px dashed yellow"; // Đánh dấu đang chọn
            });
        }

        // BẺ CONG ĐƯỜNG (Bezier)
        window.curveLine = function(layerId, color) {
            const source = map.getSource(layerId);
            const data = source._data; // Lấy GeoJSON hiện tại
            const line = turf.lineString(data.geometry.coordinates);
            
            // Tạo đường cong Bezier
            const curved = turf.bezierSpline(line, {resolution: 10000, sharpness: 0.85});
            
            source.setData(curved); // Update bản đồ
            // Update lại đầu mũi tên cho khớp hướng mới? (Khá phức tạp, tạm thời giữ nguyên vị trí cuối)
        }

        // --- 5. SỰ KIỆN BÀN PHÍM (A/D ĐỂ XOAY) ---
        document.addEventListener('keydown', (e) => {
            // Hủy vẽ
            if(e.key === "Escape" && isDrawing) stopDrawing();

            // Xoay Mũi Tên
            if(selectedArrowHead) {
                if (e.key === 'a' || e.key === 'A') {
                    selectedArrowHead.bearing -= 5;
                } else if (e.key === 'd' || e.key === 'D') {
                    selectedArrowHead.bearing += 5;
                }
                
                // Cập nhật CSS Rotate
                // Lưu ý: Icon location-arrow mặc định hướng 45 độ, nên phải trừ 45
                const icon = selectedArrowHead.element.querySelector('i');
                icon.style.transform = `rotate(${selectedArrowHead.bearing - 45}deg)`;
            }
        });

        // --- 6. HỆ THỐNG DỮ LIỆU ---
        function saveFeature(type, data, style, label) {
            mapData.features.push({type, data, style, label});
        }
        
        function addToLegend(type, color, label, icon) {
            // Check trùng
            const exists = mapData.legend.find(i => i.label === label && i.color === color);
            if(exists) return;

            mapData.legend.push({type, color, label, icon});
            
            const div = document.createElement('div');
            div.style.display = 'flex'; div.style.alignItems = 'center'; div.style.marginBottom = '5px';
            
            let symbol = '';
            if(type === 'arrow') symbol = `<div style="width:20px;height:3px;background:${color};margin-right:8px;"></div>`;
            else if(type === 'polygon') symbol = `<div style="width:20px;height:10px;background:${color};margin-right:8px;border:1px solid #000;"></div>`;
            else symbol = `<i class="fas fa-${icon}" style="color:${color};width:20px;margin-right:8px;text-align:center;"></i>`;

            div.innerHTML = symbol + `<span>${label}</span>`;
            
            const container = document.getElementById('legend-content');
            container.appendChild(div);
            document.getElementById('legend-overlay').style.display = 'block';
        }

        // --- 7. UI HELPER ---
        function toggleSidebar() {
            document.getElementById('sidebar').classList.toggle('collapsed');
        }

        function updateUI() {
            const mode = document.getElementById('draw-mode').value;
            document.getElementById('options-panel').style.display = mode === 'none' ? 'none' : 'block';
            document.getElementById('icon-options').style.display = mode === 'marker' ? 'block' : 'none';
            document.getElementById('text-options').style.display = mode === 'text' ? 'block' : 'none';
        }

        // JSON IMPORT/EXPORT (Tương tự code cũ)
        function exportJSON() {
            const dataStr = JSON.stringify(mapData);
            const blob = new Blob([dataStr], {type: "application/json"});
            const url = URL.createObjectURL(blob);
            const link = document.createElement('a');
            link.href = url; link.download = "map_lamson.json";
            link.click();
        }

        function importJSON(input) {
            const file = input.files[0];
            const reader = new FileReader();
            reader.onload = (e) => {
                const data = JSON.parse(e.target.result);
                // Load lại từng feature (Giản lược cho demo)
                alert("Đã load file thành công! (Tính năng render lại từ file cần reload trang và parse kỹ hơn)");
                console.log(data);
            };
            reader.readAsText(file);
        }

    </script>
</body>
</html>
