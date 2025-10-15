<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>배드민턴 스매싱 분석 웹페이지</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/html2canvas/1.4.1/html2canvas.min.js"></script>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        #container { display: flex; flex-direction: column; align-items: center; }
        #videoContainer { position: relative; max-width: 100%; width: 100%; }
        video { width: 100%; height: auto; }
        canvas { position: absolute; top: 0; left: 0; width: 100%; height: auto; }
        #results { margin-top: 20px; text-align: center; background: #f0f0f0; padding: 10px; border-radius: 5px; }
        input[type="file"] { margin-bottom: 10px; }
        button { padding: 10px; margin: 5px; }
        #loading { display: none; color: blue; font-weight: bold; }
        ul { text-align: left; }
        #ranking { margin-top: 30px; text-align: center; }
        #rankingList { list-style: none; padding: 0; }
        #rankingList li { margin: 5px 0; }
        #userForm { display: none; margin-top: 20px; }
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
        <button id="startButton">분석 시작</button>
        <div id="loading">분석 중...</div>
        
        <div id="videoContainer">
            <video id="video" autoplay muted playsinline></video>
            <canvas id="output_canvas"></canvas>
        </div>
        
        <div id="results" style="display: none;">
            <h2>분석 결과</h2>
            <p>자세 점수: <span id="postureScore">0</span>/10</p>
            <p>추정 스매싱 속도: <span id="speed">0</span> km/h</p>
            <p id="feedback"></p>
            <button id="downloadButton">이미지 다운로드</button>
        </div>

        <div id="userForm">
            <h3>랭킹 등록</h3>
            <input type="text" id="nickname" placeholder="닉네임" required />
            <input type="text" id="instagram" placeholder="인스타그램 ID (예: @user)" required />
            <button id="saveScoreButton">등록</button>
        </div>

        <div id="ranking">
            <h2>랭킹</h2>
            <ul id="rankingList"></ul>
        </div>
    </div>

    <script type="module">
        import {
            PoseLandmarker,
            FilesetResolver,
            DrawingUtils
        } from "https://cdn.skypack.dev/@mediapipe/tasks-vision@0.10.0";
        import { initializeApp } from "https://www.gstatic.com/firebasejs/10.14.1/firebase-app.js";
        import { getDatabase, ref, push, onValue } from "https://www.gstatic.com/firebasejs/10.14.1/firebase-database.js";

        // Firebase 초기화
        const firebaseConfig = {
            apiKey: "AIzaSyCuv6H6jyABFX3jpBoA13PLMz5hxI_Pbt8",
            authDomain: "badmi-581a6.firebaseapp.com",
            databaseURL: "https://badmi-581a6-default-rtdb.firebaseio.com",
            projectId: "badmi-581a6",
            storageBucket: "badmi-581a6.firebasestorage.app",
            messagingSenderId: "272556457679",
            appId: "1:272556457679:web:3326840f45d7ca2048c1ba",
            measurementId: "G-CNMTT9RMY8"
        };
        const app = initializeApp(firebaseConfig);
        const db = getDatabase(app);

        let poseLandmarker;
        let videoElement;
        let canvasElement;
        let canvasCtx;
        let drawingUtils;
        let isAnalyzing = false;
        let lastWristPos = null;
        let lastVideoTime = -1;
        let currentScore = 0;

        // html2canvas 로드 확인
        if (typeof html2canvas === 'undefined') {
            console.error('html2canvas is not loaded. Check CDN.');
            alert('이미지 저장 기능 로드 실패. 콘솔을 확인하세요.');
        } else {
            console.log('html2canvas loaded successfully');
        }

        document.getElementById('videoUpload').addEventListener('change', function(e) {
            const file = e.target.files[0];
            if (file && file.size > 100 * 1024 * 1024) {
                alert('파일 크기는 100MB 이하여야 합니다.');
                e.target.value = '';
            }
        });

        document.getElementById('startButton').addEventListener('click', () => {
            console.log('Start button clicked, calling startAnalysis');
            startAnalysis();
        });

        document.getElementById('downloadButton').addEventListener('click', () => {
            console.log('Download button clicked');
            downloadImage();
        });

        document.getElementById('saveScoreButton').addEventListener('click', () => {
            console.log('Save score button clicked');
            saveScore();
        });

        async function initPoseLandmarker() {
            console.log('Starting PoseLandmarker initialization...');
            try {
                const vision = await FilesetResolver.forVisionTasks(
                    "https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.0/wasm"
                );
                poseLandmarker = await PoseLandmarker.createFromOptions(vision, {
                    baseOptions: {
                        modelAssetPath: `https://storage.googleapis.com/mediapipe-models/pose_landmarker/pose_landmarker_heavy/float16/latest/pose_landmarker_heavy.task`,
                        delegate: "GPU"
                    },
                    runningMode: "VIDEO",
                    numPoses: 1,
                    minPoseDetectionConfidence: 0.3,
                    minPosePresenceConfidence: 0.3,
                    minTrackingConfidence: 0.3,
                    outputWorldLandmarks: true
                });
            } catch (e) {
                console.error('Initialization failed:', e);
                alert('PoseLandmarker 초기화 실패. 브라우저 콘솔(F12)을 확인하거나 Chrome 최신 버전을 사용하세요.');
            }
        }

        async function startAnalysis() {
            console.log('startAnalysis called');
            document.getElementById('loading').style.display = 'block';
            await initPoseLandmarker();
            if (!poseLandmarker) {
                alert('PoseLandmarker가 초기화되지 않았습니다. 콘솔 오류를 확인하세요.');
                document.getElementById('loading').style.display = 'none';
                return;
            }
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
                canvasElement.width = videoElement.videoWidth;
                canvasElement.height = videoElement.videoHeight;
                videoElement.play().catch(e => console.error('Video play failed:', e));
                isAnalyzing = true;
                analyzeVideo();
                document.getElementById('loading').style.display = 'none';
            };
            videoElement.onended = () => { isAnalyzing = false; };
        }

        async function analyzeVideo() {
            if (!isAnalyzing || !poseLandmarker) return;

            const nowInMs = performance.now();
            if (videoElement.currentTime === lastVideoTime) {
                requestAnimationFrame(analyzeVideo);
                return;
            }
            lastVideoTime = videoElement.currentTime;

            canvasCtx.save();
            canvasCtx.clearRect(0, 0, canvasElement.width, canvasElement.height);
            canvasCtx.drawImage(videoElement, 0, 0, canvasElement.width, canvasElement.height);

            try {
                const results = await poseLandmarker.detectForVideo(videoElement, nowInMs);
                if (results.landmarks && results.landmarks.length > 0) {
                    const landmarks = results.landmarks[0];
                    drawingUtils.drawLandmarks(landmarks, {color: '#FF0000', radius: 5});
                    drawingUtils.drawConnectors(landmarks, PoseLandmarker.POSE_CONNECTIONS, {color: '#00FF00'});
                    calculateScoreAndSpeed(landmarks);
                    document.getElementById('results').style.display = 'block';
                    document.getElementById('userForm').style.display = 'block';
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
                speed = Math.round(dist * 60 * 0.036);
            }
            lastWristPos = {x: wrist.x, y: wrist.y};

            currentScore = Math.max(0, score);
            document.getElementById('postureScore').textContent = currentScore;
            document.getElementById('speed').textContent = Math.min(300, speed);
            document.getElementById('feedback').textContent = getFeedback(currentScore, speed);
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

        function downloadImage() {
            console.log('Attempting to capture image with html2canvas');
            html2canvas(document.getElementById('results')).then(canvas => {
                const link = document.createElement('a');
                link.download = 'badminton_smash_analysis.png';
                link.href = canvas.toDataURL('image/png');
                link.click();
                console.log('Image downloaded successfully');
            }).catch(e => {
                console.error('html2canvas failed:', e);
                alert('이미지 저장 실패. 콘솔을 확인하세요.');
            });
        }

        function saveScore() {
            console.log('Attempting to save score to Firebase');
            const nickname = document.getElementById('nickname').value;
            const instagram = document.getElementById('instagram').value;
            if (!nickname || !instagram) {
                alert('닉네임과 인스타그램 ID를 입력하세요.');
                return;
            }
            const data = {
                score: currentScore,
                nickname: nickname,
                instagram: instagram,
                timestamp: Date.now()
            };
            push(ref(db, 'scores'), data)
                .then(() => {
                    console.log('Score saved successfully:', data);
                    alert('등록 완료! 랭킹을 확인하세요.');
                    document.getElementById('userForm').style.display = 'none';
                })
                .catch(e => {
                    console.error('Failed to save score:', e);
                    alert('랭킹 등록 실패. 콘솔을 확인하세요.');
                });
        }

        onValue(ref(db, 'scores'), snapshot => {
            console.log('Fetching ranking data');
            const rankingList = document.getElementById('rankingList');
            rankingList.innerHTML = '';
            const rankings = [];
            snapshot.forEach(child => {
                rankings.push(child.val());
            });
            rankings.sort((a, b) => b.score - a.score).slice(0, 10).forEach((rank, index) => {
                const li = document.createElement('li');
                li.textContent = `${index + 1}위: ${rank.nickname} (${rank.instagram}) - 점수: ${rank.score}`;
                rankingList.appendChild(li);
            });
        }, error => {
            console.error('Failed to fetch rankings:', error);
        });
    </script>
</body>
</html>
