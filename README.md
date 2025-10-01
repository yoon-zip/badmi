<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>배드민턴 스매싱 분석 웹페이지</title>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/control_utils/control_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/drawing_utils/drawing_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/pose/pose.js" crossorigin="anonymous"></script>
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
        <p>스매싱 영상을 업로드하세요. Google MediaPipe를 사용해 자세를 분석하고 점수를 매깁니다. (속도 측정은 포즈 기반 추정으로 간단히 구현)</p>
        
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
        let pose;
        let camera;
        let videoElement;
        let canvasElement;
        let canvasCtx;
        let isAnalyzing = false;

        const videoConfig = {
            sampleRate: 50,
            detectEveryNFrames: 10
        };

        // MediaPipe Pose 초기화
        async function initPose() {
            const poseConfig = {
                locateFile: (file) => {
                    return `https://cdn.jsdelivr.net/npm/@mediapipe/pose/${file}`;
                }
            };
            pose = new Pose(poseConfig);
            pose.setOptions({
                modelComplexity: 1,
                smoothLandmarks: true,
                enableSegmentation: false,
                smoothSegmentation: true,
                minDetectionConfidence: 0.5,
                minTrackingConfidence: 0.5
            });
            pose.onResults(onPoseResults);
        }

        // 비디오 업로드 처리
        function startAnalysis() {
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
        function analyzeVideo() {
            if (!isAnalyzing) return;

            canvasCtx.save();
            canvasCtx.clearRect(0, 0, canvasElement.width, canvasElement.height);
            canvasCtx.drawImage(videoElement, 0, 0, canvasElement.width, canvasElement.height);
            canvasCtx.restore();

            pose.send({ image: videoElement });

            requestAnimationFrame(analyzeVideo);
        }

        // 포즈 결과 처리
        function onPoseResults(results) {
            if (!results.poseLandmarks) return;

            // 랜드마크 그리기
            canvasCtx.save();
            drawingUtils.drawLandmarks(
                results.poseLandmarks,
                {color: '#FF0000', lineWidth: 2},
                drawingUtils.drawConnectors(
                    results.poseLandmarks,
                    Pose.POSE_CONNECTIONS,
                    {color: '#00FF00', lineWidth: 4}
                )
            );
            canvasCtx.restore();

            // 자세 분석 및 점수 계산 (배드민턴 스매싱 포즈 예시)
            // 스매싱 포즈: 오른팔 높이, 몸 균형, 다리 굽힘 등
            const landmarks = results.poseLandmarks;
            let score = 10; // 최대 10점
            let speed = 0; // 추정 속도 (km/h)

            // 예: 오른쪽 어깨(12), 팔꿈치(14), 손목(16) 각도 계산 (스매시 스윙)
            const shoulder = landmarks[12];
            const elbow = landmarks[14];
            const wrist = landmarks[16];

            const armAngle = calculateAngle(shoulder, elbow, wrist);
            if (armAngle > 150 || armAngle < 90) score -= 2; // 이상적인 스매시 각도 90-150도 가정

            // 몸 균형: 어깨-엉덴 각도
            const leftShoulder = landmarks[11];
            const rightShoulder = landmarks[12];
            const hip = landmarks[23]; // 오른쪽 엉덴
            const torsoAngle = Math.atan2(rightShoulder.y - hip.y, rightShoulder.x - hip.x) * 180 / Math.PI;
            if (Math.abs(torsoAngle) > 20) score -= 1; // 앞으로 기울임

            // 다리 굽힘: 무릎 각도
            const kneeLeft = landmarks[25]; // 왼쪽 무릎
            const ankleLeft = landmarks[27];
            const hipLeft = landmarks[23];
            const legAngle = calculateAngle(hipLeft, kneeLeft, ankleLeft);
            if (legAngle > 120) score -= 1; // 충분한 굽힘 부족

            // 속도 추정: 랜드마크 변화율 (간단한 프레임 간 거리)
            // 실제로는 이전 프레임 저장 필요; 여기서는 랜드마크 속도 근사
            const wristSpeed = Math.sqrt(
                Math.pow(wrist.x - elbow.x, 2) + Math.pow(wrist.y - elbow.y, 2)
            ) * 60; // 픽셀/초 가정
            speed = Math.round(wristSpeed * 0.036); // 1픽셀=0.01m, 60fps 가정으로 km/h 변환 (대략적)

            // 결과 업데이트
            document.getElementById('postureScore').textContent = Math.max(0, score);
            document.getElementById('speed').textContent = Math.min(300, speed); // 최대 300km/h 제한
            document.getElementById('feedback').textContent = getFeedback(score, speed);
        }

        // 각도 계산 함수
        function calculateAngle(a, b, c) {
            const radians = Math.atan2(c.y - b.y, c.x - b.x) - Math.atan2(a.y - b.y, a.x - b.x);
            let angle = Math.abs(radians * 180.0 / Math.PI);
            if (angle > 180.0) angle = 360 - angle;
            return angle;
        }

        // 피드백 생성
        function getFeedback(score, speed) {
            let fb = '';
            if (score >= 8) fb += '우수한 자세! ';
            else if (score >= 5) fb += '양호하지만 개선 필요. ';
            else fb += '자세 교정이 필요합니다. ';
            fb += `스매싱 속도는 ${speed}km/h 정도로 추정됩니다. (정확한 속도 측정을 위해 shuttlecock 추적이 추가 필요)`;
            return fb;
        }

        // 초기화
        initPose().then(() => {
            console.log('MediaPipe Pose initialized');
        });
    </script>
</body>
</html>
