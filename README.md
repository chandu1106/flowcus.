<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>FLOWCUS App</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
    <script src="https://kit.fontawesome.com/a13511855e.js" crossorigin="anonymous"></script>
    <style>
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f3f4f6;
        }
        .app-container {
            max-width: 500px;
            margin: auto;
            background-color: #ffffff;
            border-radius: 1.5rem;
            box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1), 0 4px 6px -2px rgba(0, 0, 0, 0.05);
            overflow: hidden;
            display: flex;
            flex-direction: column;
            min-height: 100vh;
        }
        .main-content {
            flex-grow: 1;
            padding: 1.5rem;
            overflow-y: auto;
        }
        .tab-content {
            display: none;
            animation: fadeIn 0.5s ease-in-out;
        }
        .tab-content.active {
            display: block;
        }
        @keyframes fadeIn {
            from { opacity: 0; transform: translateY(10px); }
            to { opacity: 1; transform: translateY(0); }
        }
        .tab-button {
            transition: color 0.3s, background-color 0.3s;
        }
        .tab-button.active {
            color: #4c51bf;
            background-color: #e0e7ff;
        }
        /* Custom modal for alerts */
        #modal {
            position: fixed;
            z-index: 1000;
            left: 0;
            top: 0;
            width: 100%;
            height: 100%;
            background-color: rgba(0, 0, 0, 0.4);
            display: flex;
            justify-content: center;
            align-items: center;
        }
        .modal-content {
            background-color: #fff;
            padding: 2rem;
            border-radius: 1rem;
            box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
            max-width: 90%;
            text-align: center;
        }
        .alert-message {
            margin-bottom: 1rem;
            font-size: 1.125rem;
        }
        .alert-button {
            padding: 0.5rem 1rem;
            background-color: #4c51bf;
            color: white;
            border-radius: 0.5rem;
            cursor: pointer;
        }
    </style>
</head>
<body class="bg-gray-100 flex items-center justify-center">

    <!-- Firebase and Application Logic -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, getDoc, addDoc, setDoc, updateDoc, deleteDoc, onSnapshot, collection, query, where, serverTimestamp, getDocs } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        import { setLogLevel } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        
        // Modal for alert functionality
        function showModal(message) {
            const modalDiv = document.createElement('div');
            modalDiv.id = 'modal';
            modalDiv.innerHTML = `
                <div class="modal-content">
                    <p class="alert-message">${message}</p>
                    <button class="alert-button" onclick="document.body.removeChild(document.getElementById('modal'))">OK</button>
                </div>
            `;
            document.body.appendChild(modalDiv);
        }
        
        // --- Globals ---
        let db = null;
        let auth = null;
        let userId = null;
        let map, busMarker;
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
        const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;
        window.showModal = showModal;

        // --- Firebase Init ---
        const initFirebase = async () => {
            try {
                const app = initializeApp(firebaseConfig);
                db = getFirestore(app);
                auth = getAuth(app);
                setLogLevel('debug');

                onAuthStateChanged(auth, async (user) => {
                    if (user) {
                        userId = user.uid;
                        document.getElementById('user-id-display').textContent = `User ID: ${userId}`;
                        console.log("Authenticated with user ID:", userId);
                        await setupAppListeners();
                    } else {
                        console.log("No user signed in. Attempting anonymous sign-in...");
                        if (initialAuthToken) {
                            await signInWithCustomToken(auth, initialAuthToken);
                        } else {
                            await signInAnonymously(auth);
                        }
                    }
                });
            } catch (error) {
                console.error("Firebase initialization or auth error:", error);
                showModal("Failed to initialize the application. Please try again later.");
            }
        };

        const setupAppListeners = async () => {
            if (!db || !userId) {
                console.warn("Firestore or User ID not available yet. Retrying in 1s.");
                setTimeout(setupAppListeners, 1000);
                return;
            }

            // Public collections
            const newsCollection = collection(db, `artifacts/${appId}/public/data/news`);
            const chatCollection = collection(db, `artifacts/${appId}/public/data/chat`);
            const busLocationDoc = doc(db, `artifacts/${appId}/public/data/bus/location`);

            // Private collections for the current user
            const privateAttendanceCollection = collection(db, `artifacts/${appId}/users/${userId}/attendance`);
            const privateGradesCollection = collection(db, `artifacts/${appId}/users/${userId}/grades`);

            // 1. News Listener
            onSnapshot(newsCollection, (snapshot) => {
                const newsList = document.getElementById('news-list');
                if (!newsList) return;
                newsList.innerHTML = '';
                snapshot.forEach(doc => {
                    const data = doc.data();
                    const item = document.createElement('div');
                    item.className = 'bg-gray-100 p-4 rounded-xl mb-4';
                    item.innerHTML = `<h3 class="font-semibold text-lg text-gray-800">${data.title}</h3><p class="text-sm text-gray-600">${data.content}</p>`;
                    newsList.appendChild(item);
                });
            }, (error) => console.error("Error fetching news:", error));

            // 2. Chat Listener
            onSnapshot(chatCollection, (snapshot) => {
                const chatMessages = document.getElementById('chat-messages');
                if (!chatMessages) return;
                chatMessages.innerHTML = '';
                snapshot.docs.sort((a, b) => a.data().timestamp - b.data().timestamp).forEach(doc => {
                    const data = doc.data();
                    const isCurrentUser = data.userId === userId;
                    const messageClass = isCurrentUser ? 'bg-indigo-500 text-white self-end' : 'bg-gray-200 text-gray-800 self-start';
                    const messageElement = document.createElement('div');
                    messageElement.className = `p-3 rounded-lg max-w-xs shadow-md mb-2 ${messageClass}`;
                    messageElement.innerHTML = `
                        <p class="font-medium text-xs opacity-80 mb-1">${isCurrentUser ? 'You' : `User: ${data.userId}`}</p>
                        <p>${data.text}</p>
                    `;
                    chatMessages.appendChild(messageElement);
                });
                chatMessages.scrollTop = chatMessages.scrollHeight;
            }, (error) => console.error("Error fetching chat messages:", error));

            // 3. Attendance Listener
            onSnapshot(privateAttendanceCollection, (snapshot) => {
                const attendanceList = document.getElementById('attendance-list');
                if (!attendanceList) return;
                attendanceList.innerHTML = '';
                snapshot.forEach(doc => {
                    const data = doc.data();
                    const item = document.createElement('div');
                    item.className = 'flex justify-between items-center bg-gray-100 p-4 rounded-xl mb-2';
                    item.innerHTML = `
                        <div>
                            <p class="font-semibold">${data.date}</p>
                            <p class="text-sm text-gray-600">${data.status}</p>
                        </div>
                        <button class="bg-red-500 text-white p-2 rounded-lg text-sm" onclick="deleteAttendance('${doc.id}')">Delete</button>
                    `;
                    attendanceList.appendChild(item);
                });
            }, (error) => console.error("Error fetching attendance:", error));

            // 4. Grades Listener
            onSnapshot(privateGradesCollection, (snapshot) => {
                const gradesList = document.getElementById('grades-list');
                if (!gradesList) return;
                gradesList.innerHTML = '';
                snapshot.forEach(doc => {
                    const data = doc.data();
                    const gradeColor = data.grade >= 90 ? 'text-green-600' : data.grade >= 70 ? 'text-yellow-600' : 'text-red-600';
                    const item = document.createElement('div');
                    item.className = 'flex justify-between items-center bg-gray-100 p-4 rounded-xl mb-2';
                    item.innerHTML = `
                        <div>
                            <p class="font-semibold">${data.assignment}</p>
                            <p class="text-sm text-gray-600">${data.subject}</p>
                        </div>
                        <p class="font-bold text-lg ${gradeColor}">${data.grade}%</p>
                    `;
                    gradesList.appendChild(item);
                });
            }, (error) => console.error("Error fetching grades:", error));

            // 5. Bus Tracking Listener
            onSnapshot(busLocationDoc, (doc) => {
                const location = doc.data();
                if (map && busMarker && location) {
                    const newPosition = { lat: location.lat, lng: location.lng };
                    busMarker.position = newPosition;
                    map.setCenter(newPosition);
                    document.getElementById('bus-location-info').textContent = `Bus is at: ${newPosition.lat.toFixed(4)}, ${newPosition.lng.toFixed(4)}`;
                }
            }, (error) => console.error("Error fetching bus location:", error));
            
            // Initial data population for demonstration if collections are empty
            const newsCount = (await getDocs(newsCollection)).size;
            if (newsCount === 0) {
                 await addDoc(newsCollection, { title: "Important: Parent-Teacher Meeting", content: "A virtual Parent-Teacher meeting will be held on Oct 25th. Please check your email for details.", timestamp: serverTimestamp() });
                 await addDoc(newsCollection, { title: "School Holiday", content: "School will be closed on Nov 1st for a public holiday.", timestamp: serverTimestamp() });
            }

            const attendanceCount = (await getDocs(privateAttendanceCollection)).size;
            if (attendanceCount === 0) {
                 await addDoc(privateAttendanceCollection, { date: "2025-10-21", status: "Present" });
                 await addDoc(privateAttendanceCollection, { date: "2025-10-20", status: "Present" });
                 await addDoc(privateAttendanceCollection, { date: "2025-10-19", status: "Absent" });
            }
            
            const gradesCount = (await getDocs(privateGradesCollection)).size;
            if (gradesCount === 0) {
                 await addDoc(privateGradesCollection, { assignment: "Science Fair Project", subject: "Science", grade: 95 });
                 await addDoc(privateGradesCollection, { assignment: "History Essay", subject: "History", grade: 82 });
            }
        };

        window.addAttendance = async () => {
            const date = document.getElementById('attendance-date').value;
            const status = document.getElementById('attendance-status').value;
            if (!date || !status) {
                showModal("Please select a date and status.");
                return;
            }
            try {
                const privateAttendanceCollection = collection(db, `artifacts/${appId}/users/${userId}/attendance`);
                await addDoc(privateAttendanceCollection, { date, status });
                showModal("Attendance added successfully!");
                document.getElementById('attendance-date').value = '';
            } catch (e) {
                console.error("Error adding document: ", e);
                showModal("Failed to add attendance.");
            }
        };

        window.deleteAttendance = async (docId) => {
            try {
                const privateAttendanceDoc = doc(db, `artifacts/${appId}/users/${userId}/attendance`, docId);
                await deleteDoc(privateAttendanceDoc);
                showModal("Attendance record deleted.");
            } catch (e) {
                console.error("Error deleting document: ", e);
                showModal("Failed to delete attendance record.");
            }
        };

        window.sendMessage = async () => {
            const messageInput = document.getElementById('chat-input');
            const text = messageInput.value.trim();
            if (text === '') return;
            try {
                const chatCollection = collection(db, `artifacts/${appId}/public/data/chat`);
                await addDoc(chatCollection, {
                    text,
                    userId: userId,
                    timestamp: serverTimestamp()
                });
                messageInput.value = '';
            } catch (e) {
                console.error("Error sending message: ", e);
                showModal("Failed to send message.");
            }
        };

        window.changeBusLocation = async () => {
            try {
                const busLocationDoc = doc(db, `artifacts/${appId}/public/data/bus/location`);
                const randomLat = 12.9716 + (Math.random() - 0.5) * 0.05;
                const randomLng = 77.5946 + (Math.random() - 0.5) * 0.05;
                await setDoc(busLocationDoc, { lat: randomLat, lng: randomLng });
            } catch (e) {
                console.error("Error changing bus location: ", e);
                showModal("Failed to update bus location.");
            }
        };
        
        // Google Maps Initialization
        window.initMap = async () => {
            const { Map } = await google.maps.importLibrary("maps");
            const { AdvancedMarkerElement } = await google.maps.importLibrary("marker");
            const mapOptions = {
                center: { lat: 12.9716, lng: 77.5946 }, // Default center: Bengaluru
                zoom: 12,
                mapId: 'DEMO_MAP_ID'
            };
            map = new Map(document.getElementById("map"), mapOptions);

            busMarker = new AdvancedMarkerElement({
                map: map,
                position: { lat: 12.9716, lng: 77.5946 },
                title: "School Bus",
            });
        };

        window.onload = initFirebase;
    </script>

    <div class="app-container">
        <!-- Header -->
        <header class="bg-indigo-600 text-white p-6 rounded-b-3xl shadow-md">
            <h1 class="text-3xl font-bold text-center">FLOWCUS</h1>
            <p id="user-id-display" class="text-xs text-center mt-2 opacity-75 truncate"></p>
        </header>

        <!-- Main Content Area -->
        <div class="main-content">
            <!-- News Tab -->
            <div id="news" class="tab-content active">
                <h2 class="text-2xl font-bold mb-4 text-gray-800">School News & Events</h2>
                <div id="news-list" class="space-y-4">
                    <p class="text-center text-gray-500">Loading news...</p>
                </div>
            </div>

            <!-- Attendance Tab -->
            <div id="attendance" class="tab-content">
                <h2 class="text-2xl font-bold mb-4 text-gray-800">Attendance Tracker</h2>
                <div class="bg-gray-200 p-4 rounded-xl mb-6">
                    <h3 class="font-semibold text-lg mb-2">Add New Record</h3>
                    <input type="date" id="attendance-date" class="w-full p-2 mb-2 rounded-lg" />
                    <select id="attendance-status" class="w-full p-2 mb-2 rounded-lg">
                        <option value="Present">Present</option>
                        <option value="Absent">Absent</option>
                        <option value="Late">Late</option>
                    </select>
                    <button onclick="addAttendance()" class="w-full bg-indigo-600 text-white p-3 rounded-xl font-bold">Add Attendance</button>
                </div>
                <div id="attendance-list" class="space-y-2">
                    <p class="text-center text-gray-500">Loading attendance records...</p>
                </div>
            </div>

            <!-- Grades Tab -->
            <div id="grades" class="tab-content">
                <h2 class="text-2xl font-bold mb-4 text-gray-800">Grades & Assignments</h2>
                <div id="grades-list" class="space-y-2">
                    <p class="text-center text-gray-500">Loading grades...</p>
                </div>
            </div>

            <!-- Communication Tab -->
            <div id="communication" class="tab-content flex flex-col h-full">
                <h2 class="text-2xl font-bold mb-4 text-gray-800">Teacher-Student Chat</h2>
                <div id="chat-messages" class="flex flex-col flex-grow bg-gray-100 p-4 rounded-xl overflow-y-auto mb-4">
                    <p class="text-center text-gray-500">Loading messages...</p>
                </div>
                <div class="flex">
                    <input type="text" id="chat-input" class="flex-grow p-3 rounded-l-xl border border-gray-300 focus:outline-none focus:ring-2 focus:ring-indigo-500" placeholder="Type a message..." />
                    <button onclick="sendMessage()" class="bg-indigo-600 text-white p-3 rounded-r-xl font-bold">
                        <i class="fas fa-paper-plane"></i>
                    </button>
                </div>
            </div>

            <!-- Bus Tracking Tab -->
            <div id="bus-tracking" class="tab-content">
                <h2 class="text-2xl font-bold mb-4 text-gray-800">School Bus Tracking</h2>
                <div id="map" class="h-80 w-full rounded-xl shadow-md mb-4"></div>
                <p id="bus-location-info" class="text-center text-gray-600 text-sm mb-4">Loading bus location...</p>
                <button onclick="changeBusLocation()" class="w-full bg-indigo-600 text-white p-3 rounded-xl font-bold">Simulate Bus Movement</button>
            </div>
        </div>

        <!-- Navigation Bar -->
        <nav class="bg-white p-4 flex justify-around items-center border-t border-gray-200">
            <button class="tab-button active flex flex-col items-center" onclick="showTab('news')">
                <i class="fas fa-newspaper text-xl mb-1"></i>
                <span class="text-xs">News</span>
            </button>
            <button class="tab-button flex flex-col items-center" onclick="showTab('attendance')">
                <i class="fas fa-calendar-check text-xl mb-1"></i>
                <span class="text-xs">Attendance</span>
            </button>
            <button class="tab-button flex flex-col items-center" onclick="showTab('grades')">
                <i class="fas fa-graduation-cap text-xl mb-1"></i>
                <span class="text-xs">Grades</span>
            </button>
            <button class="tab-button flex flex-col items-center" onclick="showTab('communication')">
                <i class="fas fa-comments text-xl mb-1"></i>
                <span class="text-xs">Chat</span>
            </button>
            <button class="tab-button flex flex-col items-center" onclick="showTab('bus-tracking')">
                <i class="fas fa-bus text-xl mb-1"></i>
                <span class="text-xs">Bus</span>
            </button>
        </nav>
    </div>

    <!-- Google Maps API Script -->
    <!-- NOTE: This URL requires you to provide a valid API key. Replace 'YOUR_GOOGLE_MAPS_API_KEY' with your key. -->
    <script async defer
        src="https://maps.googleapis.com/maps/api/js?key=YOUR_GOOGLE_MAPS_API_KEY&callback=initMap&libraries=marker&solution_channel=GMP_CCS_marker_v1">
    </script>
    
    <script>
        function showTab(tabId) {
            document.querySelectorAll('.tab-content').forEach(tab => tab.classList.remove('active'));
            document.getElementById(tabId).classList.add('active');

            document.querySelectorAll('.tab-button').forEach(btn => btn.classList.remove('active'));
            document.querySelector(`button[onclick="showTab('${tabId}')"]`).classList.add('active');

            if (tabId === 'bus-tracking' && typeof initMap === 'function') {
                window.initMap();
            }
        }
        
        // Initial tab display
        window.addEventListener('load', () => {
            showTab('news');
        });
    </script>
</body>
</html>

