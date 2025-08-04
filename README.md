<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Shared Photo Gallery</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    <style>
        body { padding: 20px; background-color: #f8f9fa; }
        #camera { background: #000; max-width: 100%; border-radius: 8px; }
        #photoPreview { max-width: 100%; display: none; border-radius: 8px; }
        .photo-card { position: relative; margin-bottom: 20px; }
        .photo-card img { border-radius: 8px; }
        .delete-btn { 
            position: absolute; 
            top: 10px; 
            right: 10px; 
            background: rgba(255,0,0,0.7); 
            color: white;
            border: none;
            border-radius: 50%;
            width: 30px;
            height: 30px;
            display: flex;
            align-items: center;
            justify-content: center;
        }
        .share-link {
            background: #e9ecef;
            padding: 10px;
            border-radius: 4px;
            word-break: break-all;
            margin-bottom: 20px;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1 class="text-center mb-4">Shared Photo Gallery</h1>
        
        <div class="card mb-4">
            <div class="card-body">
                <h5 class="card-title">Capture New Photo</h5>
                <div class="mb-3">
                    <video id="camera" autoplay playsinline class="w-100"></video>
                    <img id="photoPreview" class="w-100">
                </div>
                <button id="captureBtn" class="btn btn-primary me-2">Take Photo</button>
                <button id="retakeBtn" class="btn btn-secondary me-2" disabled>Retake</button>
                <button id="uploadBtn" class="btn btn-success" disabled>Upload</button>
            </div>
        </div>

        <div class="card mb-4">
            <div class="card-body">
                <h5 class="card-title">Share This Gallery</h5>
                <div class="share-link" id="shareLink"></div>
                <button id="copyLinkBtn" class="btn btn-outline-primary">Copy Link</button>
            </div>
        </div>

        <h2 class="mb-3">Gallery</h2>
        <div id="gallery" class="row"></div>
    </div>

    <!-- Firebase SDK -->
    <script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-storage-compat.js"></script>
    
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
    <script>
        // Replace with your Firebase config
        const firebaseConfig = {
            apiKey: "YOUR_API_KEY",
            authDomain: "YOUR_AUTH_DOMAIN",
            projectId: "YOUR_PROJECT_ID",
            storageBucket: "YOUR_STORAGE_BUCKET",
            messagingSenderId: "YOUR_SENDER_ID",
            appId: "YOUR_APP_ID"
        };

        // Initialize Firebase
        firebase.initializeApp(firebaseConfig);
        const db = firebase.firestore();
        const storage = firebase.storage();

        // DOM Elements
        const camera = document.getElementById('camera');
        const photoPreview = document.getElementById('photoPreview');
        const captureBtn = document.getElementById('captureBtn');
        const retakeBtn = document.getElementById('retakeBtn');
        const uploadBtn = document.getElementById('uploadBtn');
        const gallery = document.getElementById('gallery');
        const shareLink = document.getElementById('shareLink');
        const copyLinkBtn = document.getElementById('copyLinkBtn');

        let photoData = null;
        let stream = null;

        // Display shareable link
        shareLink.textContent = window.location.href;

        // Copy link button
        copyLinkBtn.addEventListener('click', () => {
            navigator.clipboard.writeText(window.location.href);
            alert('Link copied to clipboard!');
        });

        // Start Camera
        function startCamera() {
            navigator.mediaDevices.getUserMedia({ video: true })
                .then(s => {
                    stream = s;
                    camera.srcObject = stream;
                })
                .catch(err => {
                    console.error("Camera Error: ", err);
                    alert("Could not access camera. Please check permissions.");
                });
        }

        // Stop Camera
        function stopCamera() {
            if (stream) {
                stream.getTracks().forEach(track => track.stop());
            }
        }

        // Capture Photo
        captureBtn.addEventListener('click', () => {
            const canvas = document.createElement('canvas');
            canvas.width = camera.videoWidth;
            canvas.height = camera.videoHeight;
            const context = canvas.getContext('2d');
            context.drawImage(camera, 0, 0, canvas.width, canvas.height);
            
            photoData = canvas.toDataURL('image/jpeg');
            photoPreview.src = photoData;
            photoPreview.style.display = 'block';
            camera.style.display = 'none';
            
            captureBtn.disabled = true;
            retakeBtn.disabled = false;
            uploadBtn.disabled = false;
            
            stopCamera();
        });

        // Retake Photo
        retakeBtn.addEventListener('click', () => {
            photoPreview.style.display = 'none';
            camera.style.display = 'block';
            photoData = null;
            
            captureBtn.disabled = false;
            retakeBtn.disabled = true;
            uploadBtn.disabled = true;
            
            startCamera();
        });

        // Upload to Firebase
        uploadBtn.addEventListener('click', () => {
            if (!photoData) return;
            
            uploadBtn.disabled = true;
            uploadBtn.textContent = 'Uploading...';
            
            const timestamp = Date.now();
            const storageRef = storage.ref(`photos/${timestamp}.jpg`);
            
            fetch(photoData)
                .then(res => res.blob())
                .then(blob => {
                    storageRef.put(blob).then(() => {
                        storageRef.getDownloadURL().then(url => {
                            db.collection('photos').add({ 
                                url, 
                                timestamp,
                                createdAt: firebase.firestore.FieldValue.serverTimestamp() 
                            }).then(() => {
                                photoPreview.style.display = 'none';
                                camera.style.display = 'block';
                                photoData = null;
                                
                                captureBtn.disabled = false;
                                retakeBtn.disabled = true;
                                uploadBtn.disabled = true;
                                uploadBtn.textContent = 'Upload';
                                
                                startCamera();
                            });
                        });
                    }).catch(err => {
                        console.error("Upload failed:", err);
                        alert("Upload failed. Please try again.");
                        uploadBtn.disabled = false;
                        uploadBtn.textContent = 'Upload';
                    });
                });
        });

        // Delete photo
        function deletePhoto(id, imagePath) {
            if (confirm('Are you sure you want to delete this photo?')) {
                // Delete from Firestore
                db.collection('photos').doc(id).delete()
                    .then(() => {
                        // Delete from Storage
                        const storageRef = storage.refFromURL(imagePath);
                        return storageRef.delete();
                    })
                    .catch(err => {
                        console.error("Error deleting photo:", err);
                        alert("Could not delete photo.");
                    });
            }
        }

        // Load Gallery
        db.collection('photos').orderBy('createdAt', 'desc').onSnapshot(snapshot => {
            gallery.innerHTML = '';
            
            if (snapshot.empty) {
                gallery.innerHTML = '<p class="text-muted">No photos yet. Capture and upload your first photo!</p>';
                return;
            }
            
            snapshot.forEach(doc => {
                const photo = doc.data();
                const col = document.createElement('div');
                col.className = 'col-md-4 col-sm-6';
                
                col.innerHTML = `
                    <div class="photo-card">
                        <img src="${photo.url}" class="img-fluid" alt="Gallery photo">
                        <button class="delete-btn" data-id="${doc.id}" data-path="${photo.url}">
                            ×
                        </button>
                    </div>
                `;
                
                gallery.appendChild(col);
            });
            
            // Add delete event listeners
            document.querySelectorAll('.delete-btn').forEach(btn => {
                btn.addEventListener('click', (e) => {
                    deletePhoto(e.target.dataset.id, e.target.dataset.path);
                });
            });
        });

        // Initialize
        startCamera();
    </script>
</body>
</html>
