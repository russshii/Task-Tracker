import React, { useState, useEffect, useCallback, useMemo } from 'react';
import { initializeApp } from 'firebase/app';
import {
  getAuth,
  signInAnonymously,
  signInWithCustomToken,
  onAuthStateChanged,
  setPersistence, // Import setPersistence
  browserLocalPersistence // Import browserLocalPersistence
} from 'firebase/auth';
import {
  getFirestore,
  collection,
  addDoc,
  updateDoc,
  deleteDoc,
  doc,
  onSnapshot,
  query,
  orderBy,
  serverTimestamp,
  getDoc,
} from 'firebase/firestore';
import { Plus, Edit, Trash2, Save, XCircle, Download } from 'lucide-react'; // Added Download icon

// Helper function to format date for display
const formatDate = (dateString) => {
  if (!dateString) return '';
  const date = new Date(dateString);
  return date.toLocaleDateString('en-US', {
    month: 'short',
    day: '2-digit',
    year: 'numeric',
  });
};

// Function to get current time in HH:MM format
const getCurrentTime = () => {
  const now = new Date();
  const hours = now.getHours().toString().padStart(2, '0');
  const minutes = now.getMinutes().toString().padStart(2, '0');
  return `${hours}:${minutes}`;
};

// Function to get current date in YYYY-MM-DD format
const getCurrentDate = () => {
  const now = new Date();
  const year = now.getFullYear();
  const month = (now.getMonth() + 1).toString().padStart(2, '0');
  const day = now.getDate().toString().padStart(2, '0');
  return `${year}-${month}-${day}`;
};


// Predefined Task Types with shortcuts
const taskTypesList = [
  { name: 'Life Limited Parts Status', shortcut: 'LLP' },
  { name: 'Folio 12', shortcut: 'FOLIO12' },
  { name: 'Movement Traceability Sheet', shortcut: 'MTS' },
  { name: 'Airbus Aircraft Inspection Report', shortcut: 'AIR' },
  { name: 'Boeing Aircraft Readiness Log', shortcut: 'ARL' },
  { name: 'Boeing Aircraft Readiness Log Cover Page', shortcut: 'ARL COVER PAGE' },
  { name: 'Engine Data Submittal', shortcut: 'EDS' },
  { name: 'MTS LLP Hybrid', shortcut: 'MTS HYBRID' },
  { name: 'Disk Sheet', 'shortcut': 'DISK SHEET' },
  { name: 'Non-Incident Statement', shortcut: 'NIS' },
  { name: 'Airworthiness Certificate', shortcut: 'AWD' },
  { name: 'Industry Item List', shortcut: 'P&W Industry Item List' },
];

// Main App Component
const App = () => {
  const [trackingEntries, setTrackingEntries] = useState([]);
  const [formData, setFormData] = useState({
    date: getCurrentDate(), // Set initial date to current date
    timeSlot: getCurrentTime(), // Set initial time to current time
    taskBatch: '',
    assetName: '', // New state for Asset Name
    taskType: '', // Initial task type is empty, will be retained after first selection
    totalPages: '',
    errorsFound: '',
    notesChallenges: '',
  });
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState('');
  const [db, setDb] = useState(null);
  const [auth, setAuth] = useState(null);
  const [userId, setUserId] = useState(null);
  const [isAuthReady, setIsAuthReady] = useState(false);
  const [editingId, setEditingId] = useState(null);

  // Initialize Firebase and set up authentication
  useEffect(() => {
    try {
      const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
      const app = initializeApp(firebaseConfig);
      const firestoreDb = getFirestore(app);
      const firebaseAuth = getAuth(app);

      setDb(firestoreDb);
      setAuth(firebaseAuth);

      const unsubscribeAuth = onAuthStateChanged(firebaseAuth, async (user) => {
        if (user) {
          // If a user is already signed in (from __initial_auth_token or previous anonymous session), use their UID
          setUserId(user.uid);
          setIsAuthReady(true);
        } else {
          const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;
          try {
            if (initialAuthToken) {
              // Prefer signing in with the provided custom token
              await signInWithCustomToken(firebaseAuth, initialAuthToken);
            } else {
              // If no custom token, sign in anonymously and set persistence
              // Firebase Auth by default tries to persist anonymous sessions, but explicitly setting
              // browserLocalPersistence makes it clearer and potentially more robust.
              await setPersistence(firebaseAuth, browserLocalPersistence);
              const userCredential = await signInAnonymously(firebaseAuth);
              setUserId(userCredential.user.uid);
              setIsAuthReady(true);
            }
          } catch (signInError) {
            console.error('Error during Firebase sign-in:', signInError);
            setError('Failed to sign in. Please try again.');
          }
        }
      });

      return () => unsubscribeAuth();
    } catch (err) {
      console.error('Error initializing Firebase:', err);
      setError('Failed to initialize Firebase. Check your configuration.');
      setLoading(false);
    }
  }, []);

  // Fetch data from Firestore once authenticated
  useEffect(() => {
    if (!db || !userId || !isAuthReady) return;

    const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
    const trackingCollectionRef = collection(db, `artifacts/${appId}/users/${userId}/dailyTracking`);

    const q = query(trackingCollectionRef, orderBy('timestamp', 'asc'));

    const unsubscribe = onSnapshot(q, (snapshot) => {
      const entries = snapshot.docs.map(doc => ({
        id: doc.id,
        ...doc.data(),
        date: doc.data().date instanceof Date ? doc.data().date.toISOString().split('T')[0] : doc.data().date,
      }));
      setTrackingEntries(entries);
      setLoading(false);
    }, (err) => {
      console.error('Error fetching data from Firestore:', err);
      setError('Failed to load tracking data.');
      setLoading(false);
    });

    return () => unsubscribe();
  }, [db, userId, isAuthReady]);

  // Handle form input changes
  const handleChange = (e) => {
    const { name, value } = e.target;
    setFormData((prevData) => ({ ...prevData, [name]: value }));
  };

  // Clear form data and set current time/date, retaining last task type
  const clearForm = useCallback(() => {
    setFormData((prevData) => ({
      date: getCurrentDate(), // Set live date for new entries
      timeSlot: getCurrentTime(), // Set live time for new entries
      taskBatch: '',
      assetName: '', // Clear asset name on form clear
      taskType: prevData.taskType, // Retain the last selected task type
      totalPages: '',
      errorsFound: '',
      notesChallenges: '',
    }));
    setEditingId(null);
  }, []);

  // Set initial time and date when the component mounts or when form is cleared if not in editing mode
  useEffect(() => {
    setFormData(prevData => ({
      ...prevData,
      date: prevData.date || getCurrentDate(), // Set current date if not already set
      timeSlot: prevData.timeSlot || getCurrentTime(), // Set current time if not already set
    }));
  }, []);


  // Handle form submission (add or update)
  const handleSubmit = async (e) => {
    e.preventDefault();
    if (!db || !userId) {
      setError('Firestore not initialized or user not authenticated.');
      return;
    }
    setLoading(true);
    setError('');

    const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
    const trackingCollectionRef = collection(db, `artifacts/${appId}/users/${userId}/dailyTracking`);

    try {
      const entryToSave = {
        ...formData,
        timestamp: serverTimestamp(),
        totalPages: formData.totalPages ? parseInt(formData.totalPages) : 0, // Ensure totalPages is a number
        errorsFound: formData.errorsFound ? parseInt(formData.errorsFound) : 0, // Ensure errorsFound is a number
      };

      if (editingId) {
        const docRef = doc(db, `artifacts/${appId}/users/${userId}/dailyTracking`, editingId);
        const docSnap = await getDoc(docRef);

        if (docSnap.exists()) {
          await updateDoc(docRef, entryToSave);
          console.log('Document updated with ID:', editingId);
        } else {
          await addDoc(trackingCollectionRef, entryToSave);
          console.log('Document not found, added as new entry.');
        }
      } else {
        await addDoc(trackingCollectionRef, entryToSave);
        console.log('New document added.');
      }
      clearForm();
    } catch (err) {
      console.error('Error adding/updating document:', err);
      setError('Failed to save data. Please try again.');
    } finally {
      setLoading(false);
    }
  };

  // Set form data for editing an existing entry
  const handleEdit = (entry) => {
    setFormData({
      date: entry.date,
      timeSlot: entry.timeSlot,
      taskBatch: entry.taskBatch,
      assetName: entry.assetName || '', // Populate asset name for editing
      taskType: entry.taskType,
      totalPages: entry.totalPages,
      errorsFound: entry.errorsFound,
      notesChallenges: entry.notesChallenges,
    });
    setEditingId(entry.id);
  };

  // Delete an entry
  const handleDelete = async (id) => {
    if (!db || !userId) {
      setError('Firestore not initialized or user not authenticated.');
      return;
    }
    setLoading(true);
    setError('');

    const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';

    try {
      await deleteDoc(doc(db, `artifacts/${appId}/users/${userId}/dailyTracking`, id));
      console.log('Document deleted with ID:', id);
    } catch (err) {
      console.error('Error deleting document:', err);
      setError('Failed to delete data. Please try again.');
    } finally {
      setLoading(false);
    }
  };

  // Calculate totals for summary (overall)
  const totalPagesCompletedOverall = trackingEntries.reduce((sum, entry) => sum + (entry.totalPages || 0), 0);
  const totalTasksCompletedOverall = trackingEntries.length;


  // Group entries by date and then by batch within each date, calculating totals
  const groupedEntries = useMemo(() => {
    const groups = {};
    trackingEntries.forEach(entry => {
      const dateKey = entry.date;
      const batchKey = entry.taskBatch || 'No Batch'; // Handle empty batch gracefully

      if (!groups[dateKey]) {
        groups[dateKey] = {
          dailyTotalPages: 0,
          dailyErrorsFound: 0,
          entries: [], // Initialize entries array for the day
          batches: {}, // Nested object for batches
        };
      }

      if (!groups[dateKey].batches[batchKey]) {
        groups[dateKey].batches[batchKey] = {
          entries: [], // Initialize entries array for batch
          totalPages: 0,
          errorsFound: 0,
        };
      }

      // Push entry to the specific batch's entries array
      groups[dateKey].batches[batchKey].entries.push(entry);
      // Accumulate batch totals
      groups[dateKey].batches[batchKey].totalPages += (entry.totalPages || 0);
      groups[dateKey].batches[batchKey].errorsFound += (entry.errorsFound || 0);

      // Also push to daily entries for direct rendering
      groups[dateKey].entries.push(entry);

      // Accumulate daily totals directly from the entry
      groups[dateKey].dailyTotalPages += (entry.totalPages || 0);
      groups[dateKey].dailyErrorsFound += (entry.errorsFound || 0);
    });

    // Sort entries within each daily group and then within each batch
    Object.keys(groups).forEach(dateKey => {
      // Sort daily entries by timestamp
      groups[dateKey].entries.sort((a, b) => {
        const timestampA = a.timestamp?.seconds || 0;
        const timestampB = b.timestamp?.seconds || 0;
        return timestampA - timestampB;
      });
      // Sort entries within each batch by timestamp
      Object.keys(groups[dateKey].batches).forEach(batchKey => {
        groups[dateKey].batches[batchKey].entries.sort((a, b) => {
          const timestampA = a.timestamp?.seconds || 0;
          const timestampB = b.timestamp?.seconds || 0;
          return timestampA - timestampB;
        });
      });
    });


    // Convert outer object to array and sort by date
    return Object.keys(groups)
      .sort((a, b) => new Date(a) - new Date(b))
      .map(dateKey => ({
        date: dateKey,
        dailyTotalPages: groups[dateKey].dailyTotalPages,
        dailyErrorsFound: groups[dateKey].dailyErrorsFound,
        entries: groups[dateKey].entries, // Keep daily entries for direct rendering
        batches: Object.keys(groups[dateKey].batches) // Keep batches for calculation if needed, but not direct rendering of batch totals
          .sort()
          .map(batchKey => ({
            batchName: batchKey,
            totalPages: groups[dateKey].batches[batchKey].totalPages,
            errorsFound: groups[dateKey].batches[batchKey].errorsFound,
            entries: groups[dateKey].batches[batchKey].entries,
          })),
      }));
  }, [trackingEntries]);

  // Function to download data as CSV
  const downloadCSV = () => {
    if (trackingEntries.length === 0) {
      setError('No data to download.');
      return;
    }

    const headers = ["Date", "Time Slot", "Task/Batch", "Asset Name", "Task Type", "Total Pages", "Errors Found", "Notes/Challenges"]; // Added "Asset Name"
    // Map entries to CSV rows, quoting fields that might contain commas or newlines
    const rows = trackingEntries.map(entry => {
      const values = [
        formatDate(entry.date),
        entry.timeSlot,
        entry.taskBatch,
        entry.assetName || '', // Include assetName, default to empty string
        entry.taskType,
        entry.totalPages,
        entry.errorsFound,
        `"${(entry.notesChallenges || '').replace(/"/g, '""')}"` // Double quotes for notes, if notes contain quotes
      ];
      return values.join(',');
    });

    // Add daily total rows to CSV
    groupedEntries.forEach(dayGroup => {
      const totalRow = [
        `TOTAL FOR ${formatDate(dayGroup.date).toUpperCase()}`,
        '', '', '', '', // Added empty string for Asset Name column
        dayGroup.dailyTotalPages,
        dayGroup.dailyErrorsFound,
        ''
      ];
      rows.push(totalRow.join(','));
    });


    const csvContent = [headers.join(','), ...rows].join('\n');
    const blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8;' });
    const link = document.createElement('a');
    if (link.download !== undefined) { // Feature detection for download attribute
      const url = URL.createObjectURL(blob);
      link.setAttribute('href', url);
      link.setAttribute('download', `DailyTaskTracker_${getCurrentDate()}.csv`);
      link.style.visibility = 'hidden';
      document.body.appendChild(link);
      link.click();
      document.body.removeChild(link);
      URL.revokeObjectURL(url);
    } else {
      // Fallback for browsers that don't support download attribute
      // Using a custom modal/message box for alerts instead of window.alert()
      // You would implement a modal state and component to show this message
      setError('Your browser does not support downloading files directly. Please copy the data from the table.');
    }
  };


  if (loading && !isAuthReady) {
    return (
      <div className="min-h-screen flex items-center justify-center bg-gray-100 dark:bg-gray-900 text-gray-900 dark:text-gray-100">
        <div className="flex flex-col items-center p-6 bg-white dark:bg-gray-800 rounded-lg shadow-xl animate-pulse">
          <svg className="animate-spin h-10 w-10 text-blue-500" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
            <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4"></circle>
            <path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
          </svg>
          <p className="mt-4 text-lg">Loading Application...</p>
        </div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 dark:from-gray-950 dark:to-gray-850 p-4 font-inter text-gray-900 dark:text-gray-100">
      <style>
        {`
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&display=swap');
        body { font-family: 'Inter', sans-serif; }
        `}
      </style>

      {/* Header */}
      <header className="mb-8 text-center">
        <h1 className="text-4xl md:text-5xl font-extrabold text-blue-800 dark:text-blue-400 mb-2 drop-shadow-lg">
          Daily Task Tracker
        </h1>
        <p className="text-lg text-gray-600 dark:text-gray-300">
          Effortlessly update and track your daily work progress.
        </p>
        {userId && (
          <p className="text-sm text-gray-500 dark:text-gray-400 mt-2">
            User ID: {userId}
          </p>
        )}
      </header>

      {/* Error Message */}
      {error && (
        <div className="bg-red-100 dark:bg-red-900 border border-red-400 dark:border-red-700 text-red-700 dark:text-red-300 px-4 py-3 rounded-md relative mb-4 shadow-md" role="alert">
          <strong className="font-bold">Error!</strong>
          <span className="block sm:inline ml-2">{error}</span>
        </div>
      )}

      {/* Add/Edit Entry Form */}
      <section className="bg-white dark:bg-gray-800 rounded-xl shadow-2xl p-6 md:p-8 mb-8">
        <h2 className="text-2xl font-semibold mb-6 text-blue-700 dark:text-blue-300">
          {editingId ? 'Edit Entry' : 'Add New Entry'}
        </h2>
        <form onSubmit={handleSubmit} className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
          {/* Date */}
          <div>
            <label htmlFor="date" className="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-1">Date</label>
            <input
              type="date"
              id="date"
              name="date"
              value={formData.date}
              onChange={handleChange}
              className="mt-1 block w-full px-3 py-2 border border-gray-300 dark:border-gray-600 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm bg-gray-50 dark:bg-gray-700 text-gray-900 dark:text-gray-100"
              required
            />
          </div>
          {/* Time Slot */}
          <div>
            <label htmlFor="timeSlot" className="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-1">Time Slot</label>
            <input
              type="time"
              id="timeSlot"
              name="timeSlot"
              value={formData.timeSlot}
              onChange={handleChange}
              className="mt-1 block w-full px-3 py-2 border border-gray-300 dark:border-gray-600 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm bg-gray-50 dark:bg-gray-700 text-gray-900 dark:text-gray-100"
            />
          </div>
          {/* Task/Batch */}
          <div>
            <label htmlFor="taskBatch" className="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-1">Task/Batch</label>
            <input
              type="text"
              id="taskBatch"
              name="taskBatch"
              value={formData.taskBatch}
              onChange={handleChange}
              placeholder="e.g., 17419"
              className="mt-1 block w-full px-3 py-2 border border-gray-300 dark:border-gray-600 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm bg-gray-50 dark:bg-gray-700 text-gray-900 dark:text-gray-100"
            />
          </div>
          {/* Asset Name - New Input Field */}
          <div>
            <label htmlFor="assetName" className="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-1">Asset Name</label>
            <input
              type="text"
              id="assetName"
              name="assetName"
              value={formData.assetName}
              onChange={handleChange}
              placeholder="e.g., Engine XYZ"
              className="mt-1 block w-full px-3 py-2 border border-gray-300 dark:border-gray-600 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm bg-gray-50 dark:bg-gray-700 text-gray-900 dark:text-gray-100"
            />
          </div>
          {/* Task Type - Dropdown with predefined list */}
          <div>
            <label htmlFor="taskType" className="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-1">Task Type</label>
            <select
              id="taskType"
              name="taskType"
              value={formData.taskType}
              onChange={handleChange}
              className="mt-1 block w-full px-3 py-2 border border-gray-300 dark:border-gray-600 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm bg-gray-50 dark:bg-gray-700 text-gray-900 dark:text-gray-100"
            >
              <option value="">Select a Task Type</option>
              {taskTypesList.map((type, index) => (
                <option key={index} value={type.shortcut}>
                  {type.name} – {type.shortcut}
                </option>
              ))}
            </select>
          </div>
          {/* Total Pages */}
          <div>
            <label htmlFor="totalPages" className="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-1">Total Pages</label>
            <input
              type="number"
              id="totalPages"
              name="totalPages"
              value={formData.totalPages}
              onChange={handleChange}
              placeholder="e.g., 43"
              className="mt-1 block w-full px-3 py-2 border border-gray-300 dark:border-gray-600 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm bg-gray-50 dark:bg-gray-700 text-gray-900 dark:text-gray-100"
            />
          </div>
          {/* Errors Found */}
          <div>
            <label htmlFor="errorsFound" className="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-1">Errors Found</label>
            <input
              type="number"
              id="errorsFound"
              name="errorsFound"
              value={formData.errorsFound}
              onChange={handleChange}
              placeholder="e.g., 2"
              className="mt-1 block w-full px-3 py-2 border border-gray-300 dark:border-gray-600 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm bg-gray-50 dark:bg-gray-700 text-gray-900 dark:text-gray-100"
            />
          </div>
          {/* Notes/Challenges */}
          <div className="col-span-1 md:col-span-2 lg:col-span-3">
            <label htmlFor="notesChallenges" className="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-1">Notes/Challenges</label>
            <textarea
              id="notesChallenges"
              name="notesChallenges"
              value={formData.notesChallenges}
              onChange={handleChange}
              rows="3"
              placeholder="Any notes or challenges faced during the task..."
              className="mt-1 block w-full px-3 py-2 border border-gray-300 dark:border-gray-600 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm bg-gray-50 dark:bg-gray-700 text-gray-900 dark:text-gray-100"
            ></textarea>
          </div>

          {/* Form Actions */}
          <div className="col-span-1 md:col-span-2 lg:col-span-3 flex justify-end gap-4 mt-4">
            {editingId && (
              <button
                type="button"
                onClick={clearForm}
                className="inline-flex items-center px-4 py-2 border border-transparent text-sm font-medium rounded-md shadow-sm text-gray-700 bg-gray-200 hover:bg-gray-300 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-gray-500 transition-colors duration-200 dark:bg-gray-600 dark:text-gray-200 dark:hover:bg-gray-700"
              >
                <XCircle className="mr-2 h-5 w-5" />
                Cancel Edit
              </button>
            )}
            <button
              type="submit"
              className="inline-flex items-center px-4 py-2 border border-transparent text-sm font-medium rounded-md shadow-sm text-white bg-blue-600 hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-blue-500 transition-colors duration-200 transform hover:scale-105 active:scale-95"
              disabled={loading}
            >
              {loading ? (
                <svg className="animate-spin h-5 w-5 text-white mr-2" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                  <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4"></circle>
                  <path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
                </svg>
              ) : editingId ? (
                <>
                  <Save className="mr-2 h-5 w-5" />
                  Save Changes
                </>
              ) : (
                <>
                  <Plus className="mr-2 h-5 w-5" />
                  Add Entry
                </>
              )}
            </button>
          </div>
        </form>
      </section>

      {/* Tracking Summary (Overall) */}
      <section className="bg-white dark:bg-gray-800 rounded-xl shadow-2xl p-6 md:p-8 mb-8">
        <h2 className="text-2xl font-semibold mb-6 text-blue-700 dark:text-blue-300">
          Overall Progress Summary
        </h2>
        <div className="grid grid-cols-1 md:grid-cols-2 gap-4 text-center">
          <div className="bg-blue-50 dark:bg-blue-900/20 p-4 rounded-lg shadow-sm">
            <p className="text-xl font-bold text-blue-700 dark:text-blue-300">{totalPagesCompletedOverall}</p>
            <p className="text-sm text-gray-600 dark:text-gray-400">Total Pages Completed</p>
          </div>
          <div className="bg-green-50 dark:bg-green-900/20 p-4 rounded-lg shadow-sm">
            <p className="text-xl font-bold text-green-700 dark:text-green-300">{totalTasksCompletedOverall}</p>
            <p className="text-sm text-gray-600 dark:text-gray-400">Total Tasks Completed</p>
          </div>
        </div>
        <div className="mt-6 text-center">
          <button
            onClick={downloadCSV}
            className="inline-flex items-center px-6 py-3 border border-transparent text-base font-medium rounded-md shadow-sm text-white bg-blue-500 hover:bg-blue-600 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-blue-500 transition-colors duration-200 transform hover:scale-105 active:scale-95"
          >
            <Download className="mr-3 h-5 w-5" />
            Download Data as CSV
          </button>
        </div>
      </section>

      {/* Tracking Table */}
      <section className="bg-white dark:bg-gray-800 rounded-xl shadow-2xl p-6 md:p-8">
        <h2 className="text-2xl font-semibold mb-6 text-blue-700 dark:text-blue-300">
          Your Tracking History
        </h2>
        {trackingEntries.length === 0 && !loading && (
          <p className="text-center text-gray-500 dark:text-gray-400">No entries yet. Add your first task above!</p>
        )}

        {loading && (
          <div className="flex justify-center items-center py-8">
            <svg className="animate-spin h-8 w-8 text-blue-500" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
              <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4"></circle>
              <path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
            </svg>
            <span className="ml-3 text-lg text-gray-600 dark:text-gray-300">Loading entries...</span>
          </div>
        )}

        {!loading && trackingEntries.length > 0 && (
          <div className="overflow-x-auto rounded-lg border border-gray-200 dark:border-gray-700 shadow-md">
            <table className="min-w-full divide-y divide-gray-200 dark:divide-gray-700">
              <thead className="bg-gray-50 dark:bg-gray-700">
                <tr>
                  <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 dark:text-gray-300 uppercase tracking-wider">Date</th>
                  <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 dark:text-gray-300 uppercase tracking-wider">Time Slot</th>
                  <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 dark:text-gray-300 uppercase tracking-wider">Task/Batch</th>
                  <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 dark:text-gray-300 uppercase tracking-wider">Asset Name</th> {/* New Header */}
                  <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 dark:text-gray-300 uppercase tracking-wider">Task Type</th>
                  <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 dark:text-gray-300 uppercase tracking-wider">Total Pages</th>
                  <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 dark:text-gray-300 uppercase tracking-wider">Errors Found</th>
                  <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 dark:text-gray-300 uppercase tracking-wider">Notes/Challenges</th>
                  <th scope="col" className="relative px-6 py-3">
                    <span className="sr-only">Actions</span>
                  </th>
                </tr>
              </thead>
              {/* Using a function to render tbody content to prevent whitespace text nodes */}
              <tbody className="bg-white dark:bg-gray-800 divide-y divide-gray-200 dark:divide-gray-700">{
                groupedEntries.map((dayGroup) => (
                  <React.Fragment key={dayGroup.date}>
                    {dayGroup.entries.map((entry) => (
                      <tr key={entry.id} className="hover:bg-gray-50 dark:hover:bg-gray-700 transition-colors duration-150">
                        <td className="px-6 py-4 whitespace-nowrap text-sm font-medium text-gray-900 dark:text-gray-100">{formatDate(entry.date)}</td>
                        <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-500 dark:text-gray-300">{entry.timeSlot}</td>
                        <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-500 dark:text-gray-300">{entry.taskBatch}</td>
                        <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-500 dark:text-gray-300">{entry.assetName}</td> {/* Display Asset Name */}
                        <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-500 dark:text-gray-300">{entry.taskType}</td>
                        <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-500 dark:text-gray-300">{entry.totalPages}</td>
                        <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-500 dark:text-gray-300">{entry.errorsFound}</td>
                        <td className="px-6 py-4 text-sm text-gray-500 dark:text-gray-300 max-w-xs overflow-hidden text-ellipsis">{entry.notesChallenges}</td>
                        <td className="px-6 py-4 whitespace-nowrap text-right text-sm font-medium">
                          <div className="flex justify-end space-x-2">
                            <button
                              onClick={() => handleEdit(entry)}
                              className="text-blue-600 hover:text-blue-900 dark:text-blue-400 dark:hover:text-blue-200 p-1 rounded-md hover:bg-blue-100 dark:hover:bg-blue-900 transition-colors duration-150"
                              title="Edit Entry"
                            >
                              <Edit className="h-5 w-5" />
                            </button>
                            <button
                              onClick={() => handleDelete(entry.id)}
                              className="text-red-600 hover:text-red-900 dark:text-red-400 dark:hover:text-red-200 p-1 rounded-md hover:bg-red-100 dark:hover:bg-red-900 transition-colors duration-150"
                              title="Delete Entry"
                            >
                              <Trash2 className="h-5 w-5" />
                            </button>
                          </div>
                        </td>
                      </tr>
                    ))}
                    {/* Daily Total Row */}
                    <tr className="bg-blue-100 dark:bg-blue-900/40 font-bold text-blue-800 dark:text-blue-200">
                      <td className="px-6 py-3 whitespace-nowrap text-sm" colSpan="4">TOTAL FOR {formatDate(dayGroup.date).toUpperCase()}</td>
                      <td className="px-6 py-3 whitespace-nowrap text-sm">{dayGroup.dailyTotalPages}</td>
                      <td className="px-6 py-3 whitespace-nowrap text-sm">{dayGroup.dailyErrorsFound}</td>
                      <td className="px-6 py-3 whitespace-nowrap text-sm" colSpan="2"></td>
                    </tr>
                  </React.Fragment>
                ))
              }</tbody>
            </table>
          </div>
        )}
      </section>

      {/* Footer - Optional */}
      <footer className="mt-8 text-center text-gray-500 dark:text-gray-400 text-sm">
        <p>&copy; {new Date().getFullYear()} Daily Task Tracker. All rights reserved.</p>
      </footer>
    </div>
  );
};

export default App;
