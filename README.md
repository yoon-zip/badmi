<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>배드민턴 스매싱 분석 웹페이지</title>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.0"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/drawing_utils/drawing_utils.js" crossorigin="anonymous"></script>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        #container { display: flex; flex-direction: column; align-items: center; }
        #videoContainer { position: relative; }
        canvas { position: absolute; top: 0; left: 0; }
        #results { margin-top: 20px; text-align: center; }
        input[type="file"] { margin-bottom: 10px; }
        button { padding: 10px; margin: 5px; }
        #loading { display: none; color: blue; font-weight: bold; }
        ul { text-align: left; }
    </style>
</head>
<body>
    <div id="container">
        <h1>배드민턴 스매싱 분석기</h1>
        <p>최적 분석을 위해:</p>
        <ul>
            <li>720p 이상, 30fps 이상으로 촬영</li>
            <li>밝은 조명, 어두운 배경 사용</li>
            <li>측면 또는 45도 각도에서 스매싱 동작 촬영</li>
            <li>MP4, MOV, WebM 형식, 100MB 이하, 5~10초 영상</li>
        </ul>
        
        <input type="file" id="videoUpload" accept="video/mp4,video/quicktime,video/webm" />
        <button onclick="startAnalysis()">분석 시작</button>
        <div id="loading">분석 중...</div>
        
        <div id="videoContainer">
            <video id="video" autoplay muted playsinline></video>
            <canvas id="output_canvas"></canvas>
        </div>
        
        <div id="results">
            <h2>분석 결과</h2>
            <p>자세 점수: <span id="postureScore">0</span>/10</p>
            <p>추정 스매싱 속도: <span id="speed">0</span> km/h</p>
            <p id="feedback"></p>
        </div>
    </div>

    <script>
        let poseLandmarker;
        let videoElement;
        let canvasElement;
        let canvasCtx;
        let drawingUtils;
        let isAnalyzing = false;
        let lastWristPos = null;

        document.getElementById('videoUpload').addEventListener('change', function(e) {
            const file = e.target.files[0];
            if (file && file.size > 100 * 1024 * 1024) {
                alert('파일 크기는 100MB 이하여야 합니다.');
                e.target.value = '';
            }
        });

        async function initPoseLandmarker() {
            console.log('Initializing PoseLandmarker...');
            try {
                const vision = await FilesetResolver.forVisionTasks(
                    "https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.0/wasm"
                );
                poseLandmarker = await PoseLandmarker.createFromOptions(vision, {
                    baseOptions: {
                        modelAssetPath: `https://storage.googleapis.com/mediapipe-models/pose_landmarker/pose_landmarker_lite/float16/1/pose_landmarker_lite.task`,
                        delegate: "CPU"
                    },
                    runningMode: "VIDEO",
                    numPoses: 1,
                    minPoseDetectionConfidence: 0.3,
                    minPosePresenceConfidence: 0.3,
                    minTrackingConfidence: 0.3,
                    outputWorldLandmarks: true
                });
                console.log('PoseLandmarker initialized successfully');
            } catch (e) {
                console.error('Initialization failed:', e);
                alert('PoseLandmarker 초기화 실패. 콘솔 오류를 확인하세요.');
            }
        }

        async function startAnalysis() {
            document.getElementById('loading').style.display = 'block';
            await initPoseLandmarker();
            const fileInput = document.getElementById('videoUpload');
            const file = fileInput.files[0];
            if (!file) {
                alert('영상을 업로드하세요.');
                document.getElementById('loading').style.display = 'none';
                return;
            }

            videoElement = document.getElementById('video');
            canvasElement = document.getElementById('output_canvas');
            canvasCtx = canvasElement.getContext('2d');
            drawingUtils = new DrawingUtils(canvasCtx);

            const videoSrc = URL.createObjectURL(file);
            videoElement.src = videoSrc;
            videoElement.onloadedmetadata = () => {
                console.log('Video loaded:', videoElement.videoWidth, videoElement.videoHeight);
                canvasElement.width = videoElement.videoWidth;
                canvasElement.height = videoElement.videoHeight;
                videoElement.play().catch(e => console.error('Video play failed:', e));
                isAnalyzing = true;
                analyzeVideo();
                document.getElementById('loading').style.display = 'none';
            };
        }

        async function analyzeVideo() {
            if (!isAnalyzing || !poseLandmarker) return;

            canvasCtx.save();
            canvasCtx.clearRect(0, 0, canvasElement.width, canvasElement.height);
            canvasCtx.drawImage(videoElement, 0, 0, canvasElement.width, canvasElement.height);

            try {
                const results = await poseLandmarker.detectForVideo(videoElement, performance.now());
                console.log('Pose results:', results);
                if (results.landmarks && results.landmarks.length > 0) {
                    const landmarks = results.landmarks[0];
                    drawingUtils.drawLandmarks(landmarks, {color: '#FF0000', radius: 5});
                    drawingUtils.drawConnectors(landmarks, PoseLandmarker.POSE_CONNECTIONS, {color: '#00FF00'});
                    calculateScoreAndSpeed(landmarks);
                } else {
                    console.warn('No landmarks detected');
                }
            } catch (e) {
                console.error('Detection failed:', e);
            }

            canvasCtx.restore();
            requestAnimationFrame(analyzeVideo);
        }

        function calculateScoreAndSpeed(landmarks) {
            let score = 10;
            let speed = 0;

            const shoulder = landmarks[12] || {x:0,y:0};
            const elbow = landmarks[14] || {x:0,y:0};
            const wrist = landmarks[16] || {x:0,y:0};
            const armAngle = calculateAngle(shoulder, elbow, wrist);
            if (armAngle > 150 || armAngle < 90) score -= 2;

            const hip = landmarks[24] || {x:0,y:0};
            const torsoAngle = Math.atan2(shoulder.y - hip.y, shoulder.x - hip.x) * 180 / Math.PI;
            if (Math.abs(torsoAngle) > 20) score -= 1;

            const hipLeft = landmarks[23] || {x:0,y:0};
            const kneeLeft = landmarks[25] || {x:0,y:0};
            const ankleLeft = landmarks[27] || {x:0,y:0};
            const legAngle = calculateAngle(hipLeft, kneeLeft, ankleLeft);
            if (legAngle > 120) score -= 1;

            if (lastWristPos) {
                const dx = wrist.x - lastWristPos.x;
                const dy = wrist.y - lastWristPos.y;
                const dist = Math.sqrt(dx*dx + dy*dy);
                speed = Math.round(dist * 60 * 0.036); // 대략 km/h (조정 가능)
            }
            lastWristPos = {x: wrist.x, y: wrist.y};

            document.getElementById('postureScore').textContent = Math.max(0, score);
            document.getElementById('speed').textContent = Math.min(300, speed);
            document.getElementById('feedback').textContent = getFeedback(score, speed);
        }

        function calculateAngle(a, b, c) {
            const ab = {x: a.x - b.x, y: a.y - b.y};
            const cb = {x: c.x - b.x, y: c.y - b.y};
            const dot = ab.x * cb.x + ab.y * cb.y;
            const modAb = Math.sqrt(ab.x * ab.x + ab.y * ab.y);
            const modCb = Math.sqrt(cb.x * cb.x + cb.y * cb.y);
            if (modAb === 0 || modCb === 0) return 0;
            return Math.acos(dot / (modAb * modCb)) * 180 / Math.PI;
        }

        function getFeedback(score, speed) {
            let fb = score >= 8 ? '우수한 자세! ' : score >= 5 ? '양호하지만 개선 필요. ' : '자세 교정이 필요합니다. ';
            fb += `스매싱 속도: ${speed}km/h (손목 움직임 추정). 팔 각도, 몸 균형, 다리 굽힘을 확인하세요.`;
            return fb;
        }
    </script>
</body>
</html>
