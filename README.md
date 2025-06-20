import React, { useState, useEffect, useRef } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, doc, setDoc, getDoc, collection, addDoc, onSnapshot, query, orderBy, serverTimestamp } from 'firebase/firestore';

// Main App Component
function App() {
    // Firebase related states
    const [db, setDb] = useState(null);
    const [auth, setAuth] = useState(null);
    const [userId, setUserId] = useState(null);

    // App specific states for navigation and data
    const [currentView, setCurrentView] = useState('profile'); // 'profile', 'discover', 'chat'
    const [userProfile, setUserProfile] = useState(null); // Current user's profile data
    const [profiles, setProfiles] = useState([]); // All other users' profiles
    const [globalChatMessages, setGlobalChatMessages] = useState([]); // Messages for global chat
    const [newChatMessage, setNewChatMessage] = useState(''); // Input for new chat message

    // Global variables from Canvas environment
    const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
    const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
    const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

    const chatMessagesEndRef = useRef(null); // Ref for scrolling global chat

    // Effect for initializing Firebase and handling authentication
    useEffect(() => {
        const initFirebase = async () => {
            try {
                // Initialize Firebase app
                const app = initializeApp(firebaseConfig);
                const firestoreDb = getFirestore(app);
                const firebaseAuth = getAuth(app);

                setDb(firestoreDb);
                setAuth(firebaseAuth);

                // Sign in with custom token if available, otherwise anonymously
                if (initialAuthToken) {
                    await signInWithCustomToken(firebaseAuth, initialAuthToken);
                } else {
                    await signInAnonymously(firebaseAuth);
                }

                // Listen for auth state changes to set user ID
                onAuthStateChanged(firebaseAuth, (user) => {
                    if (user) {
                        setUserId(user.uid);
                        console.log("User signed in:", user.uid);
                    } else {
                        // For anonymous users, generate a UUID as a pseudo-user ID
                        // Note: This ID will change if the user clears browser data or on new anonymous session
                        // For persistent anonymous users, a different strategy (e.g., storing UUID in local storage) might be needed.
                        setUserId(crypto.randomUUID());
                        console.log("User signed out or anonymous.");
                    }
                });

            } catch (error) {
                console.error("Error initializing Firebase or signing in:", error);
            }
        };

        initFirebase();
    }, [initialAuthToken, firebaseConfig]);

    // Effect to fetch current user's profile and other profiles
    useEffect(() => {
        if (db && userId) {
            // Fetch current user's profile
            const userProfileDocRef = doc(db, `/artifacts/${appId}/public/data/profiles`, userId);
            const unsubscribeProfile = onSnapshot(userProfileDocRef, (docSnap) => {
                if (docSnap.exists()) {
                    setUserProfile({ id: docSnap.id, ...docSnap.data() });
                } else {
                    setUserProfile(null); // Profile does not exist yet
                }
            }, (error) => {
                console.error("Error fetching user profile:", error);
            });

            // Fetch all other profiles for discover section
            const profilesCollectionRef = collection(db, `/artifacts/${appId}/public/data/profiles`);
            const unsubscribeProfiles = onSnapshot(profilesCollectionRef, (snapshot) => {
                const fetchedProfiles = [];
                snapshot.forEach((doc) => {
                    if (doc.id !== userId) { // Exclude current user's profile
                        fetchedProfiles.push({ id: doc.id, ...doc.data() });
                    }
                });
                setProfiles(fetchedProfiles);
            }, (error) => {
                console.error("Error fetching other profiles:", error);
            });

            // Clean up listeners
            return () => {
                unsubscribeProfile();
                unsubscribeProfiles();
            };
        }
    }, [db, userId, appId]);

    // Effect to fetch global chat messages
    useEffect(() => {
        if (db && userId) {
            const globalChatCollectionPath = `/artifacts/${appId}/public/data/global_chat_messages`;
            const chatCollectionRef = collection(db, globalChatCollectionPath);
            const q = query(chatCollectionRef, orderBy("timestamp", "asc"));

            const unsubscribeChat = onSnapshot(q, (snapshot) => {
                const fetchedMessages = [];
                snapshot.forEach((doc) => {
                    fetchedMessages.push({ id: doc.id, ...doc.data() });
                });
                setGlobalChatMessages(fetchedMessages);
            }, (error) => {
                console.error("Error fetching global chat messages:", error);
            });

            return () => unsubscribeChat();
        }
    }, [db, userId, appId]);

    // Effect to scroll global chat to bottom on new messages
    useEffect(() => {
        scrollToChatBottom();
    }, [globalChatMessages]);

    const scrollToChatBottom = () => {
        chatMessagesEndRef.current?.scrollIntoView({ behavior: "smooth" });
    };

    // --- Profile Management Functions ---
    const handleSaveProfile = async (profileData) => {
        if (!db || !userId) {
            console.error("Database not initialized or user not logged in.");
            return;
        }
        try {
            const profileDocRef = doc(db, `/artifacts/${appId}/public/data/profiles`, userId);
            await setDoc(profileDocRef, {
                ...profileData,
                timestamp: serverTimestamp(), // Add/update timestamp
            }, { merge: true }); // Merge to update existing fields or create if not exists
            alert("Profile saved successfully!"); // Use a custom modal in real app
        } catch (error) {
            console.error("Error saving profile:", error);
            alert("Error saving profile: " + error.message); // Use a custom modal
        }
    };

    // --- Global Chat Functions ---
    const handleSendChatMessage = async () => {
        if (newChatMessage.trim() === '' || !db || !userId || !userProfile) {
            console.log("Cannot send message: Empty, DB/User not ready, or Profile missing.");
            return;
        }
        try {
            const chatCollectionRef = collection(db, `/artifacts/${appId}/public/data/global_chat_messages`);
            await addDoc(chatCollectionRef, {
                userId: userId,
                userName: userProfile.name || `User ${userId.substring(0, 5)}`, // Use name from profile
                text: newChatMessage,
                timestamp: serverTimestamp(),
            });
            setNewChatMessage('');
        } catch (error) {
            console.error("Error sending global chat message:", error);
        }
    };

    // --- Helper Components for Views ---

    // Profile Component
    const ProfileComponent = ({ userProfile, onSaveProfile, userId, isNewProfile }) => {
        const [name, setName] = useState(userProfile?.name || '');
        const [bio, setBio] = useState(userProfile?.bio || '');
        const [interests, setInterests] = useState(userProfile?.interests?.join(', ') || '');

        useEffect(() => {
            if (userProfile) {
                setName(userProfile.name || '');
                setBio(userProfile.bio || '');
                setInterests(userProfile.interests?.join(', ') || '');
            }
        }, [userProfile]);

        const handleSubmit = (e) => {
            e.preventDefault();
            const interestArray = interests.split(',').map(item => item.trim()).filter(item => item !== '');
            onSaveProfile({ name, bio, interests: interestArray });
        };

        return (
            <div className="p-6 bg-white rounded-xl shadow-lg m-4 w-full max-w-md">
                <h2 className="text-2xl font-bold text-blue-700 mb-6 text-center">
                    {isNewProfile ? "Create Your Profile" : "Edit Your Profile"}
                </h2>
                <div className="text-sm text-gray-600 mb-4 text-center">
                    Your Public ID: <span className="font-mono bg-gray-100 px-2 py-1 rounded-md">{userId}</span>
                </div>
                <form onSubmit={handleSubmit} className="space-y-4">
                    <div>
                        <label htmlFor="name" className="block text-sm font-medium text-gray-700 mb-1">Name</label>
                        <input
                            type="text"
                            id="name"
                            className="w-full p-3 border border-gray-300 rounded-lg focus:ring-blue-500 focus:border-blue-500"
                            value={name}
                            onChange={(e) => setName(e.target.value)}
                            placeholder="Your Name"
                            required
                        />
                    </div>
                    <div>
                        <label htmlFor="bio" className="block text-sm font-medium text-gray-700 mb-1">Bio</label>
                        <textarea
                            id="bio"
                            className="w-full p-3 border border-gray-300 rounded-lg focus:ring-blue-500 focus:border-blue-500 h-24 resize-none"
                            value={bio}
                            onChange={(e) => setBio(e.target.value)}
                            placeholder="Tell us about yourself..."
                        ></textarea>
                    </div>
                    <div>
                        <label htmlFor="interests" className="block text-sm font-medium text-gray-700 mb-1">Interests (comma-separated)</label>
                        <input
                            type="text"
                            id="interests"
                            className="w-full p-3 border border-gray-300 rounded-lg focus:ring-blue-500 focus:border-blue-500"
                            value={interests}
                            onChange={(e) => setInterests(e.target.value)}
                            placeholder="e.g., hiking, reading, cooking"
                        />
                    </div>
                    <button
                        type="submit"
                        className="w-full bg-blue-600 hover:bg-blue-700 text-white font-bold py-3 rounded-full shadow-lg transition-all duration-300 transform hover:scale-105 active:scale-95"
                    >
                        Save Profile
                    </button>
                </form>
            </div>
        );
    };

    // Discover Component
    const DiscoverComponent = ({ profiles }) => {
        return (
            <div className="p-4 w-full flex-1 overflow-y-auto">
                <h2 className="text-3xl font-bold text-center text-blue-700 mb-6">Discover Other Users</h2>
                {profiles.length === 0 ? (
                    <div className="text-center text-gray-500 text-lg mt-10">
                        No other profiles found yet. Be the first to create one!
                    </div>
                ) : (
                    <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
                        {profiles.map((profile) => (
                            <div key={profile.id} className="bg-white rounded-xl shadow-md p-6">
                                <h3 className="text-xl font-semibold text-gray-800 mb-2">{profile.name || `User ${profile.id.substring(0, 8)}...`}</h3>
                                <p className="text-gray-600 text-sm mb-3 italic">{profile.bio || "No bio provided."}</p>
                                {profile.interests && profile.interests.length > 0 && (
                                    <div className="flex flex-wrap gap-2 mb-3">
                                        {profile.interests.map((interest, idx) => (
                                            <span key={idx} className="bg-blue-100 text-blue-700 text-xs font-medium px-2.5 py-0.5 rounded-full">
                                                {interest}
                                            </span>
                                        ))}
                                    </div>
                                )}
                                <div className="text-xs text-gray-500 mt-2">
                                    User ID: <span className="font-mono">{profile.id.substring(0, 8)}...</span>
                                </div>
                            </div>
                        ))}
                    </div>
                )}
            </div>
        );
    };

    // Global Chat Component
    const GlobalChatComponent = ({ messages, newChatMessage, setNewChatMessage, handleSendChatMessage, chatMessagesEndRef, userId }) => {
        return (
            <div className="flex flex-col flex-1 w-full p-4 bg-gray-50 rounded-xl shadow-lg m-4">
                <h2 className="text-3xl font-bold text-center text-blue-700 mb-4">Global Chat</h2>
                <div className="flex-1 overflow-y-auto p-4 bg-white rounded-lg border border-gray-200 shadow-inner space-y-4 mb-4">
                    {messages.length === 0 ? (
                        <div className="text-center text-gray-500 mt-10">No messages yet. Say hello to the world!</div>
                    ) : (
                        messages.map((msg) => (
                            <div
                                key={msg.id}
                                className={`flex ${msg.userId === userId ? 'justify-end' : 'justify-start'}`}
                            >
                                <div
                                    className={`rounded-xl p-3 max-w-xs sm:max-w-md shadow-md ${
                                        msg.userId === userId
                                            ? 'bg-blue-500 text-white rounded-br-none'
                                            : 'bg-gray-200 text-gray-800 rounded-bl-none'
                                    }`}
                                >
                                    <div className="font-semibold text-sm mb-1">
                                        {msg.userId === userId ? 'You' : `${msg.userName || `User ${msg.userId.substring(0, 8)}`}`}
                                    </div>
                                    <p className="text-sm break-words">{msg.text}</p>
                                    {msg.timestamp && (
                                        <div className="text-xs mt-1 opacity-75 text-right">
                                            {new Date(msg.timestamp.toDate()).toLocaleTimeString()}
                                        </div>
                                    )}
                                </div>
                            </div>
                        ))
                    )}
                    <div ref={chatMessagesEndRef} />
                </div>
                <div className="flex items-center space-x-3">
                    <input
                        type="text"
                        className="flex-1 p-3 border border-gray-300 rounded-full focus:outline-none focus:ring-2 focus:ring-blue-500"
                        placeholder="Type your message to the world..."
                        value={newChatMessage}
                        onChange={(e) => setNewChatMessage(e.target.value)}
                        onKeyPress={(e) => e.key === 'Enter' && handleSendChatMessage()}
                        disabled={!userId || !userProfile} // Disable if not ready
                    />
                    <button
                        onClick={handleSendChatMessage}
                        className="bg-blue-600 hover:bg-blue-700 text-white font-bold py-3 px-6 rounded-full shadow-lg transition-all duration-300 transform hover:scale-105 active:scale-95 disabled:opacity-50 disabled:cursor-not-allowed"
                        disabled={!newChatMessage.trim() || !userId || !userProfile}
                    >
                        Send
                    </button>
                </div>
            </div>
        );
    };


    // Main App Render
    return (
        <div className="flex flex-col h-screen bg-gradient-to-br from-blue-100 to-indigo-200 font-inter text-gray-900">
            {/* Header and Navigation */}
            <header className="bg-white shadow-md p-4 flex flex-col sm:flex-row items-center justify-between rounded-b-2xl">
                <h1 className="text-3xl font-extrabold text-blue-800 mb-3 sm:mb-0">World Connector</h1>
                <nav className="flex space-x-4">
                    <button
                        onClick={() => setCurrentView('profile')}
                        className={`py-2 px-5 rounded-full font-medium transition-all duration-200 ${
                            currentView === 'profile' ? 'bg-blue-600 text-white shadow-md' : 'text-blue-700 hover:bg-blue-100'
                        }`}
                    >
                        My Profile
                    </button>
                    <button
                        onClick={() => setCurrentView('discover')}
                        className={`py-2 px-5 rounded-full font-medium transition-all duration-200 ${
                            currentView === 'discover' ? 'bg-blue-600 text-white shadow-md' : 'text-blue-700 hover:bg-blue-100'
                        }`}
                    >
                        Discover
                    </button>
                    <button
                        onClick={() => setCurrentView('chat')}
                        className={`py-2 px-5 rounded-full font-medium transition-all duration-200 ${
                            currentView === 'chat' ? 'bg-blue-600 text-white shadow-md' : 'text-blue-700 hover:bg-blue-100'
                        }`}
                    >
                        Global Chat
                    </button>
                </nav>
            </header>

            {/* Main Content Area */}
            <main className="flex-1 flex flex-col items-center justify-center p-4">
                {!userId ? (
                    <div className="text-center text-lg text-gray-600">
                        Initializing app... Please wait.
                    </div>
                ) : (
                    <>
                        {currentView === 'profile' && (
                            <ProfileComponent
                                userProfile={userProfile}
                                onSaveProfile={handleSaveProfile}
                                userId={userId}
                                isNewProfile={!userProfile}
                            />
                        )}
                        {currentView === 'discover' && (
                            <DiscoverComponent profiles={profiles} />
                        )}
                        {currentView === 'chat' && (
                            <GlobalChatComponent
                                messages={globalChatMessages}
                                newChatMessage={newChatMessage}
                                setNewChatMessage={setNewChatMessage}
                                handleSendChatMessage={handleSendChatMessage}
                                chatMessagesEndRef={chatMessagesEndRef}
                                userId={userId}
                            />
                        )}
                    </>
                )}
            </main>

            {/* Optional: Footer */}
            <footer className="bg-gray-800 text-white text-center p-3 text-sm rounded-t-lg shadow-inner">
                <p>&copy; 2024 World Connector. All rights reserved.</p>
            </footer>
        </div>
    );
}

export default App;
