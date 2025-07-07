import React, { useState, useEffect, useRef } from 'react';
import { initializeApp } from 'firebase/app';
import { 
    getAuth, 
    signInAnonymously, 
    signInWithCustomToken, 
    onAuthStateChanged,
    setPersistence,
    browserLocalPersistence
} from 'firebase/auth';
import { 
    getFirestore, 
    collection, 
    addDoc, 
    doc, 
    updateDoc, 
    deleteDoc, 
    onSnapshot,
    serverTimestamp,
    query,
    where
} from 'firebase/firestore';

// --- Helper Functions & Configuration ---

// Safely get Firebase config and App ID from global scope
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-blogger-app';

// --- Icon Components (using SVG for simplicity) ---

const PlusIcon = () => (
    <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M5 12h14"/><path d="M12 5v14"/></svg>
);

const EditIcon = () => (
    <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M17 3a2.85 2.85 0 1 1 4 4L7.5 20.5 2 22l1.5-5.5Z"/><path d="m15 5 4 4"/></svg>
);

const TrashIcon = () => (
    <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M3 6h18"/><path d="M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6"/><path d="M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2"/><line x1="10" x2="10" y1="11" y2="17"/><line x1="14" x2="14" y1="11" y2="17"/></svg>
);

const ArrowLeftIcon = () => (
    <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="m12 19-7-7 7-7"/><path d="M19 12H5"/></svg>
);


// --- Main App Components ---

// Header Component
const Header = ({ onNewPostClick, onTitleClick, userId }) => (
    <header className="bg-gray-800/50 backdrop-blur-sm sticky top-0 z-20 border-b border-gray-700">
        <div className="container mx-auto px-4 sm:px-6 lg:px-8">
            <div className="flex items-center justify-between h-16">
                <div 
                    className="text-2xl font-bold text-cyan-400 cursor-pointer hover:text-cyan-300 transition-colors"
                    onClick={onTitleClick}
                >
                    Fire<span className="text-white">Blog</span>
                </div>
                <div className="flex items-center space-x-4">
                     {userId && (
                        <div className="hidden sm:block text-xs text-gray-400 bg-gray-700 px-3 py-1 rounded-full">
                            User ID: {userId}
                        </div>
                    )}
                    <button 
                        onClick={onNewPostClick}
                        className="flex items-center space-x-2 bg-cyan-600 hover:bg-cyan-700 text-white font-semibold px-4 py-2 rounded-lg shadow-md transition-all duration-200 transform hover:scale-105"
                    >
                        <PlusIcon />
                        <span className="hidden sm:inline">New Post</span>
                    </button>
                </div>
            </div>
            {userId && (
                <div className="sm:hidden text-center pb-2">
                    <div className="text-xs text-gray-400 bg-gray-700 px-3 py-1 rounded-full inline-block">
                        User ID: {userId}
                    </div>
                </div>
            )}
        </div>
    </header>
);

// Post List Item Component
const PostListItem = ({ post, onSelectPost }) => (
    <div 
        onClick={() => onSelectPost(post)}
        className="bg-gray-800 rounded-xl p-6 cursor-pointer hover:bg-gray-700/80 transition-all duration-300 transform hover:-translate-y-1 shadow-lg hover:shadow-cyan-500/20"
    >
        <h2 className="text-2xl font-bold text-cyan-400 mb-2">{post.title}</h2>
        <p className="text-gray-400 mb-4 line-clamp-3">{post.content}</p>
        <div className="text-xs text-gray-500">
            <span>By: {post.authorId.substring(0, 8)}...</span> | <span>{new Date(post.createdAt?.toDate()).toLocaleString()}</span>
        </div>
    </div>
);

// Post List Component
const PostList = ({ posts, onSelectPost, onNewPostClick }) => {
    // Sort posts by creation date, newest first
    const sortedPosts = [...posts].sort((a, b) => b.createdAt?.toMillis() - a.createdAt?.toMillis());

    return (
        <div className="space-y-6">
            {sortedPosts.length > 0 ? (
                sortedPosts.map(post => <PostListItem key={post.id} post={post} onSelectPost={onSelectPost} />)
            ) : (
                <div className="text-center py-16 px-6 bg-gray-800 rounded-xl">
                    <h3 className="text-xl font-semibold text-white">No posts yet!</h3>
                    <p className="text-gray-400 mt-2 mb-6">Be the first to share your thoughts.</p>
                    <button 
                        onClick={onNewPostClick}
                        className="flex items-center mx-auto space-x-2 bg-cyan-600 hover:bg-cyan-700 text-white font-semibold px-4 py-2 rounded-lg shadow-md transition-all duration-200 transform hover:scale-105"
                    >
                        <PlusIcon />
                        <span>Create First Post</span>
                    </button>
                </div>
            )}
        </div>
    );
};

// Post Page Component
const PostPage = ({ post, onBack, onEdit, onDelete, currentUserId }) => (
    <div className="bg-gray-800 rounded-xl p-6 sm:p-8 shadow-lg animate-fade-in">
        <button onClick={onBack} className="flex items-center space-x-2 text-cyan-400 hover:text-cyan-300 mb-6 font-semibold">
            <ArrowLeftIcon />
            <span>Back to Posts</span>
        </button>
        <h1 className="text-4xl font-extrabold text-white mb-4">{post.title}</h1>
        <div className="text-sm text-gray-500 mb-6 border-b border-gray-700 pb-4">
            <span>By: {post.authorId}</span> | <span>{new Date(post.createdAt?.toDate()).toLocaleString()}</span>
        </div>
        <div className="prose prose-invert max-w-none text-gray-300 whitespace-pre-wrap text-lg leading-relaxed">
            {post.content}
        </div>
        {currentUserId === post.authorId && (
            <div className="mt-8 pt-6 border-t border-gray-700 flex items-center space-x-4">
                <button onClick={() => onEdit(post)} className="flex items-center space-x-2 bg-yellow-500 hover:bg-yellow-600 text-white font-semibold px-4 py-2 rounded-lg shadow-md transition-all duration-200">
                    <EditIcon />
                    <span>Edit</span>
                </button>
                <button onClick={() => onDelete(post.id)} className="flex items-center space-x-2 bg-red-600 hover:bg-red-700 text-white font-semibold px-4 py-2 rounded-lg shadow-md transition-all duration-200">
                    <TrashIcon />
                    <span>Delete</span>
                </button>
            </div>
        )}
    </div>
);

// Post Editor Component
const PostEditor = ({ onSave, onCancel, postToEdit }) => {
    const [title, setTitle] = useState('');
    const [content, setContent] = useState('');
    const contentRef = useRef(null);

    useEffect(() => {
        if (postToEdit) {
            setTitle(postToEdit.title);
            setContent(postToEdit.content);
        }
    }, [postToEdit]);

    useEffect(() => {
        // Auto-resize textarea
        if (contentRef.current) {
            contentRef.current.style.height = 'auto';
            contentRef.current.style.height = `${contentRef.current.scrollHeight}px`;
        }
    }, [content]);

    const handleSave = () => {
        if (title.trim() && content.trim()) {
            onSave({ title, content });
            setTitle('');
            setContent('');
        }
    };

    return (
        <div className="bg-gray-800 rounded-xl p-6 sm:p-8 shadow-lg animate-fade-in space-y-6">
            <h2 className="text-3xl font-bold text-white">{postToEdit ? 'Edit Post' : 'Create New Post'}</h2>
            <input
                type="text"
                value={title}
                onChange={(e) => setTitle(e.target.value)}
                placeholder="Post Title"
                className="w-full bg-gray-700 text-white p-3 rounded-lg border-2 border-gray-600 focus:border-cyan-500 focus:ring-cyan-500 outline-none transition-colors"
            />
            <textarea
                ref={contentRef}
                value={content}
                onChange={(e) => setContent(e.target.value)}
                placeholder="Write your thoughts here..."
                className="w-full bg-gray-700 text-white p-3 rounded-lg border-2 border-gray-600 focus:border-cyan-500 focus:ring-cyan-500 outline-none transition-colors min-h-[200px] resize-none overflow-hidden"
                rows="10"
            />
            <div className="flex items-center justify-end space-x-4">
                <button onClick={onCancel} className="bg-gray-600 hover:bg-gray-500 text-white font-semibold px-6 py-2 rounded-lg transition-colors">
                    Cancel
                </button>
                <button onClick={handleSave} className="bg-cyan-600 hover:bg-cyan-700 text-white font-semibold px-6 py-2 rounded-lg transition-colors">
                    Save Post
                </button>
            </div>
        </div>
    );
};

// Confirmation Modal Component
const ConfirmationModal = ({ isOpen, message, onConfirm, onCancel }) => {
    if (!isOpen) return null;

    return (
        <div className="fixed inset-0 bg-black/60 backdrop-blur-sm flex items-center justify-center z-50 animate-fade-in">
            <div className="bg-gray-800 rounded-xl p-8 shadow-2xl max-w-sm w-full mx-4">
                <h3 className="text-xl font-bold text-white mb-4">Are you sure?</h3>
                <p className="text-gray-300 mb-6">{message}</p>
                <div className="flex justify-end space-x-4">
                    <button onClick={onCancel} className="bg-gray-600 hover:bg-gray-500 text-white font-semibold px-4 py-2 rounded-lg transition-colors">
                        Cancel
                    </button>
                    <button onClick={onConfirm} className="bg-red-600 hover:bg-red-700 text-white font-semibold px-4 py-2 rounded-lg transition-colors">
                        Confirm
                    </button>
                </div>
            </div>
        </div>
    );
};


// --- App Component (Main Logic) ---

export default function App() {
    // Firebase state
    const [db, setDb] = useState(null);
    const [auth, setAuth] = useState(null);
    const [userId, setUserId] = useState(null);
    const [isAuthReady, setIsAuthReady] = useState(false);

    // App state
    const [page, setPage] = useState('list'); // 'list', 'post', 'editor'
    const [posts, setPosts] = useState([]);
    const [selectedPost, setSelectedPost] = useState(null);
    const [editingPost, setEditingPost] = useState(null);
    const [loading, setLoading] = useState(true);
    
    // Modal state
    const [isModalOpen, setIsModalOpen] = useState(false);
    const [postToDeleteId, setPostToDeleteId] = useState(null);

    // --- Firebase Initialization and Auth ---
    useEffect(() => {
        try {
            const app = initializeApp(firebaseConfig);
            const firestoreDb = getFirestore(app);
            const firebaseAuth = getAuth(app);
            
            setDb(firestoreDb);
            setAuth(firebaseAuth);

            onAuthStateChanged(firebaseAuth, async (user) => {
                if (user) {
                    setUserId(user.uid);
                } else {
                    const token = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;
                    try {
                        if (token) {
                            await signInWithCustomToken(firebaseAuth, token);
                        } else {
                            await signInAnonymously(firebaseAuth);
                        }
                    } catch (error) {
                        console.error("Authentication failed:", error);
                    }
                }
                setIsAuthReady(true);
            });
        } catch (error) {
            console.error("Firebase initialization failed:", error);
            setIsAuthReady(true); // Proceed even if firebase fails to show an error state
        }
    }, []);

    // --- Firestore Data Fetching ---
    useEffect(() => {
        if (isAuthReady && db) {
            setLoading(true);
            const postsCollectionPath = `/artifacts/${appId}/public/data/posts`;
            const q = query(collection(db, postsCollectionPath));
            
            const unsubscribe = onSnapshot(q, (querySnapshot) => {
                const postsData = [];
                querySnapshot.forEach((doc) => {
                    postsData.push({ id: doc.id, ...doc.data() });
                });
                setPosts(postsData);
                setLoading(false);
            }, (error) => {
                console.error("Error fetching posts:", error);
                setLoading(false);
            });

            return () => unsubscribe();
        }
    }, [isAuthReady, db]);

    // --- Event Handlers ---
    const handleSelectPost = (post) => {
        setSelectedPost(post);
        setPage('post');
    };

    const handleBackToList = () => {
        setPage('list');
        setSelectedPost(null);
        setEditingPost(null);
    };

    const handleNewPostClick = () => {
        setEditingPost(null);
        setPage('editor');
    };

    const handleEditPost = (post) => {
        setEditingPost(post);
        setPage('editor');
    };

    const handleSavePost = async (postData) => {
        if (!db || !userId) return;
        
        const postsCollectionPath = `/artifacts/${appId}/public/data/posts`;

        if (editingPost) {
            // Update existing post
            const postRef = doc(db, postsCollectionPath, editingPost.id);
            await updateDoc(postRef, {
                ...postData,
            });
        } else {
            // Create new post
            await addDoc(collection(db, postsCollectionPath), {
                ...postData,
                authorId: userId,
                createdAt: serverTimestamp(),
            });
        }
        handleBackToList();
    };

    const handleDeleteClick = (id) => {
        setPostToDeleteId(id);
        setIsModalOpen(true);
    };

    const confirmDeletePost = async () => {
        if (!db || !postToDeleteId) return;
        const postsCollectionPath = `/artifacts/${appId}/public/data/posts`;
        await deleteDoc(doc(db, postsCollectionPath, postToDeleteId));
        handleBackToList();
        setIsModalOpen(false);
        setPostToDeleteId(null);
    };

    // --- Render Logic ---
    const renderPage = () => {
        if (loading) {
            return (
                <div className="text-center py-20">
                    <div className="loader inline-block w-12 h-12 border-4 border-t-cyan-400 border-gray-600 rounded-full animate-spin"></div>
                    <p className="mt-4 text-gray-400">Loading Posts...</p>
                </div>
            );
        }

        switch (page) {
            case 'post':
                return <PostPage post={selectedPost} onBack={handleBackToList} onEdit={handleEditPost} onDelete={handleDeleteClick} currentUserId={userId} />;
            case 'editor':
                return <PostEditor onSave={handleSavePost} onCancel={handleBackToList} postToEdit={editingPost} />;
            case 'list':
            default:
                return <PostList posts={posts} onSelectPost={handleSelectPost} onNewPostClick={handleNewPostClick} />;
        }
    };

    return (
        <div className="bg-gray-900 text-white min-h-screen font-sans">
            <style>{`
                .animate-fade-in { animation: fadeIn 0.5s ease-in-out; }
                @keyframes fadeIn { from { opacity: 0; transform: translateY(-10px); } to { opacity: 1; transform: translateY(0); } }
                .prose-invert h1, .prose-invert h2, .prose-invert h3 { color: white; }
                .prose-invert p, .prose-invert li { color: #d1d5db; }
                .prose-invert a { color: #22d3ee; }
                .prose-invert blockquote { border-left-color: #6b7280; }
                .prose-invert code { color: #f9fafb; }
            `}</style>
            
            <Header onNewPostClick={handleNewPostClick} onTitleClick={handleBackToList} userId={userId} />
            
            <main className="container mx-auto p-4 sm:p-6 lg:p-8">
                {renderPage()}
            </main>

            <footer className="text-center py-6 text-gray-600 text-sm">
                <p>Powered by React & Firebase</p>
                <p>App ID: {appId}</p>
            </footer>

            <ConfirmationModal 
                isOpen={isModalOpen}
                message="This action cannot be undone. The post will be permanently deleted."
                onConfirm={confirmDeletePost}
                onCancel={() => setIsModalOpen(false)}
            />
        </div>
    );
}
