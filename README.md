<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>배드민턴 스매싱 분석 웹페이지</title>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@latest/vision_bundle.js" crossorigin="anonymous"></script>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        #container { display: flex; flex-direction: column; align-items: center; }
        #videoContainer { position: relative; }
        canvas { position: absolute; top: 0; left: 0; }
        #results { margin-top: 20px; text-align: center; }
        input[type="file"] { margin-bottom: 10px; }
        button { padding: 10px; margin: 5px; }
    </style>
</head>
<body>
    <div id="container">
        <h1>배드민턴 스매싱 분석기</h1>
        <p>스매싱 영상을 업로드하세요. MediaPipe Tasks로 자세 분석 (속도 추정 포함).</p>
        
        <input type="file" id="videoUpload" accept="video/*" />
        <button onclick="startAnalysis()">분석 시작</button>
        
        <div id="videoContainer">
            <video id="video" autoplay muted></video>
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
        let isAnalyzing = false;
        let lastWristPos = null;

        // MediaPipe Pose Connections (스켈레톤 연결선 정의)
        const POSE_CONNECTIONS = [
            [11, 12], [11, 13], [13, 15], [12, 14], [14, 16], // 상체 (어깨-팔)
            [11, 23], [12, 24], [23, 24], // 어깨-엉덩이
            [23, 25], [25, 27], [24, 26], [26, 28], // 다리
        ];

        // MediaPipe Pose Landmarker 초기화
        async function initPoseLandmarker() {
            const vision = await FilesetResolver.forVisionTasks(
                "https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@latest/wasm"
            );
            poseLandmarker = await PoseLandmarker.createFromOptions(vision, {
                baseOptions: {
                    modelAssetPath: `https://storage.googleapis.com/mediapipe-models/pose_landmarker/pose_landmarker_lite/float16/1/pose_landmarker_lite.task`,
                    delegate: "GPU"
                },
                runningMode: "VIDEO",
                numPoses: 1,
                minPoseDetectionConfidence: 0.5,
                minPosePresenceConfidence: 0.5,
                minTrackingConfidence: 0.5,
                outputWorldLandmarks: true
            });
        }

        // 비디오 업로드 및 분석 시작
        async function startAnalysis() {
            await initPoseLandmarker();
            const fileInput = document.getElementById('videoUpload');
            const file = fileInput.files[0];
            if (!file) {
                alert('영상을 업로드하세요.');
                return;
            }

            videoElement = document.getElementById('video');
            canvasElement = document.getElementById('output_canvas');
            canvasCtx = canvasElement.getContext('2d');

            const videoSrc = URL.createObjectURL(file);
            videoElement.src = videoSrc;
            videoElement.onloadedmetadata = () => {
                canvasElement.width = videoElement.videoWidth;
                canvasElement.height = videoElement.videoHeight;
                videoElement.play();
                isAnalyzing = true;
                analyzeVideo();
            };
        }

        // 비디오 프레임 분석 루프
        async function analyzeVideo() {
            if (!isAnalyzing || !poseLandmarker) return;

            canvasCtx.save();
            canvasCtx.clearRect(0, 0, canvasElement.width, canvasElement.height);
            canvasCtx.drawImage(videoElement, 0, 0, canvasElement.width, canvasElement.height);

            const results = await poseLandmarker.detectForVideo(videoElement, performance.now());
            if (results.landmarks && results.landmarks.length > 0) {
                const landmarks = results.landmarks[0]; // 2D 화면 좌표
                drawLandmarksAndConnections(landmarks);
                calculateScoreAndSpeed(landmarks);
            }

            canvasCtx.restore();
            requestAnimationFrame(analyzeVideo);
        }

        // 랜드마크와 연결선 그리기
        function drawLandmarksAndConnections(landmarks) {
            // 관절 포인트 (빨간 점)
            landmarks.forEach(lm => {
                if (lm.visibility && lm.visibility < 0.5) return; // 낮은 신뢰도 포인트 제외
                canvasCtx.beginPath();
                canvasCtx.arc(lm.x * canvasElement.width, lm.y * canvasElement.height, 4, 0, 2 * Math.PI);
                canvasCtx.fillStyle = '#FF0000';
                canvasCtx.fill();
            });

            // 연결선 (녹색)
            canvasCtx.strokeStyle = '#00FF00';
            canvasCtx.lineWidth = 2;
            POSE_CONNECTIONS.forEach(([start, end]) => {
                const startLm = landmarks[start];
                const endLm = landmarks[end];
                if (startLm && endLm && startLm.visibility > 0.5 && endLm.visibility > 0.5) {
                    canvasCtx.beginPath();
                    canvasCtx.moveTo(startLm.x * canvasElement.width, startLm.y * canvasElement.height);
                    canvasCtx.lineTo(endLm.x * canvasElement.width, endLm.y * canvasElement.height);
                    canvasCtx.stroke();
                }
            });
        }

        // 자세 점수 및 속도 계산
        function calculateScoreAndSpeed(landmarks) {
            let score = 10;
            let speed = 0;

            // 팔 각도: 어깨(12), 팔꿈치(14), 손목(16)
            const shoulder = landmarks[12] || {x:0,y:0};
            const elbow = landmarks[14] || {x:0,y:0};
            const wrist = landmarks[16] || {x:0,y:0};

            const armAngle = calculateAngle(shoulder, elbow, wrist);
            if (armAngle > 150 || armAngle < 90) score -= 2;

            // 속도: 손목 위치 변화
            if (lastWristPos) {
                const dx = wrist.x - lastWristPos.x;
                const dy = wrist.y - lastWristPos.y;
                const dist = Math.sqrt(dx*dx + dy*dy);
                speed = Math.round(dist * 60 * 0.036); // 대략 km/h
            }
            lastWristPos = wrist;

            document.getElementById('postureScore').textContent = Math.max(0, score);
            document.getElementById('speed').textContent = Math.min(300, speed);
            document.getElementById('feedback').textContent = getFeedback(score, speed);
        }

        // 각도 계산 (2D)
        function calculateAngle(a, b, c) {
            const ab = {x: a.x - b.x, y: a.y - b.y};
            const cb = {x: c.x - b.x, y: c.y - b.y};
            const dot = ab.x * cb.x + ab.y * cb.y;
            const modAb = Math.sqrt(ab.x * ab.x + ab.y * ab.y);
            const modCb = Math.sqrt(cb.x * cb.x + cb.y * cb.y);
            return Math.acos(dot / (modAb * modCb)) * 180 / Math.PI;
        }

        function getFeedback(score, speed) {
            let fb = score >= 8 ? '우수한 자세! ' : score >= 5 ? '양호하지만 개선 필요. ' : '자세 교정이 필요합니다. ';
            fb += `스매싱 속도: ${speed}km/h (추정). 셔틀콕 추적 추가 추천.`;
            return fb;
        }
    </script>
</body>
</html>
