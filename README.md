import React, { useState, useEffect, useMemo } from 'react';
import { initializeApp } from 'firebase/app';
import {
  getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged,
  User
} from 'firebase/auth';
import {
  getFirestore, collection, addDoc, onSnapshot,
  query, orderBy, Timestamp, setLogLevel, QuerySnapshot,
  Firestore, Auth
} from 'firebase/firestore';

// --- Type Definitions for better code clarity ---
/**
 * @typedef {Object} Expense
 * @property {string} id - Firestore document ID
 * @property {number} amount
 * @property {string} type - 'income' or 'expense'
 * @property {string} category
 * @property {string} description
 * @property {string} date - ISO date string (YYYY-MM-DD)
 * @property {Timestamp} timestamp - Firestore server timestamp
 */

/**
 * @typedef {Object} NewExpense
 * @property {number | string} amount
 * @property {string} category
 * @property {string} type
 * @property {string} description
 * @property {string} date
 */

// Global variables provided by the Canvas environment
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : null;
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

// Categories for the tracker. Positive amounts are 'Income', negative or default are 'Expense'.
const CATEGORIES = [
  { name: 'Income', type: 'income', icon: '💸' },
  { name: 'Groceries', type: 'expense', icon: '🛒' },
  { name: 'Housing', type: 'expense', icon: '🏠' },
  { name: 'Transportation', type: 'expense', icon: '🚌' },
  { name: 'Entertainment', type: 'expense', icon: '🎬' },
  { name: 'Utilities', type: 'expense', icon: '💡' },
  { name: 'Health', type: 'expense', icon: '⚕️' },
  { name: 'Savings', type: 'expense', icon: '🏦' },
  { name: 'Other', type: 'expense', icon: '📦' },
];

/**
 * Main application component for the Personal Finance Tracker.
 * @returns {JSX.Element}
 */
const App = () => {
  // --- Firebase/Auth State ---
  const [db, setDb] = useState(null);
  const [auth, setAuth] = useState(null);
  const [userId, setUserId] = useState(null);
  const [isAuthReady, setIsAuthReady] = useState(false);
  const [authError, setAuthError] = useState('');

  // --- Application Data State ---
  /** @type {[Expense[], React.Dispatch<React.SetStateAction<Expense[]>>]} */
  const [expenses, setExpenses] = useState([]);
  const [loadingExpenses, setLoadingExpenses] = useState(true);
  const [savingExpense, setSavingExpense] = useState(false);

  // --- Form State ---
  /** @type {[NewExpense, React.Dispatch<React.SetStateAction<NewExpense>>]} */
  const [newExpense, setNewExpense] = useState({
    amount: '',
    category: CATEGORIES[1].name, // Default to Groceries
    type: CATEGORIES[1].type,
    description: '',
    date: new Date().toISOString().split('T')[0], // Today's date
  });

  // --- Firebase Initialization and Authentication Effect ---
  useEffect(() => {
    if (!firebaseConfig) {
      setAuthError("Firebase configuration is missing.");
      return;
    }

    try {
      setLogLevel('Debug');
      const app = initializeApp(firebaseConfig);
      const authInstance = getAuth(app);
      const firestoreInstance = getFirestore(app);

      setAuth(authInstance);
      setDb(firestoreInstance);

      // Listen for auth state change
      const unsubscribe = onAuthStateChanged(authInstance, async (user) => {
        if (user) {
          setUserId(user.uid);
          setIsAuthReady(true);
        } else {
          // Attempt sign-in if needed
          try {
            if (initialAuthToken) {
              await signInWithCustomToken(authInstance, initialAuthToken);
            } else {
              await signInAnonymously(authInstance);
            }
          } catch (e) {
            console.error("Firebase Auth Error:", e);
            setAuthError(`Authentication failed: ${e.message}`);
            // Fallback for UI if sign-in fails but we still need a userId for data path
            setUserId(authInstance.currentUser?.uid || crypto.randomUUID());
            setIsAuthReady(true);
          }
        }
      });

      return () => unsubscribe();
    } catch (e) {
      console.error("Firebase Initialization Error:", e);
      setAuthError(`Initialization failed: ${e.message}`);
    }
  }, []);

  // --- Real-time Expense Fetching Effect ---
  useEffect(() => {
    if (!isAuthReady || !db || !userId) return;

    setLoadingExpenses(true);
    const userExpensesPath = `/artifacts/${appId}/users/${userId}/expenses`;
    const expensesCollectionRef = collection(db, userExpensesPath);

    // Query to fetch all expenses, ordered by timestamp (newest first)
    const q = query(expensesCollectionRef, orderBy('timestamp', 'desc'));

    const unsubscribe = onSnapshot(q, (snapshot) => {
      const fetchedExpenses = snapshot.docs.map(doc => {
        /** @type {Expense} */
        const data = doc.data();
        return {
          id: doc.id,
          ...data,
          // Ensure amount is treated as a number
          amount: parseFloat(data.amount) || 0,
        };
      });
      setExpenses(fetchedExpenses);
      setLoadingExpenses(false);
    }, (error) => {
      console.error("Error listening to expenses:", error);
      setLoadingExpenses(false);
      // Optional: Display error to user
    });

    return () => unsubscribe();
  }, [isAuthReady, db, userId]);


  // --- Event Handlers ---

  /**
   * Updates the form state for a new expense.
   * @param {React.ChangeEvent<HTMLInputElement | HTMLSelectElement | HTMLTextAreaElement>} e
   */
  const handleInputChange = (e) => {
    const { name, value } = e.target;
    setNewExpense(prev => {
      const updatedExpense = { ...prev, [name]: value };

      // Auto-update expense type when category changes
      if (name === 'category') {
        const selectedCategory = CATEGORIES.find(c => c.name === value);
        updatedExpense.type = selectedCategory ? selectedCategory.type : 'expense';
      }

      return updatedExpense;
    });
  };

  /**
   * Saves a new expense document to Firestore.
   * @param {React.FormEvent} e
   */
  const handleAddExpense = async (e) => {
    e.preventDefault();
    if (!db || !userId || savingExpense) return;

    // Basic validation
    if (!newExpense.amount || !newExpense.category || !newExpense.date) {
      alert("Please fill in amount, category, and date.");
      return;
    }

    setSavingExpense(true);
    try {
      const amountValue = parseFloat(newExpense.amount);
      if (isNaN(amountValue) || amountValue <= 0) {
        alert("Amount must be a positive number.");
        setSavingExpense(false);
        return;
      }

      const expenseData = {
        amount: amountValue,
        type: newExpense.type,
        category: newExpense.category,
        description: newExpense.description || 'N/A',
        date: newExpense.date,
        timestamp: Timestamp.now(), // Server timestamp for ordering
      };

      const userExpensesPath = `/artifacts/${appId}/users/${userId}/expenses`;
      await addDoc(collection(db, userExpensesPath), expenseData);

      // Reset form after successful submission
      setNewExpense({
        amount: '',
        category: CATEGORIES[1].name,
        type: CATEGORIES[1].type,
        description: '',
        date: new Date().toISOString().split('T')[0],
      });

    } catch (e) {
      console.error("Error adding document: ", e);
      alert("Failed to save expense. See console for details.");
    } finally {
      setSavingExpense(false);
    }
  };


  // --- Reporting Logic (Memoized for performance) ---

  /**
   * Calculates monthly reports based on the current expenses array.
   * @returns {{monthKey: string, totalIncome: number, totalExpense: number, netBalance: number, categoryBreakdown: {[category: string]: number}}[]}
   */
  const monthlyReports = useMemo(() => {
    /** @type {{[key: string]: { totalIncome: number, totalExpense: number, categoryBreakdown: {[category: string]: number} }}} */
    const reports = {};

    expenses.forEach(expense => {
      // Create a key for the month (YYYY-MM)
      const monthKey = expense.date.substring(0, 7);

      if (!reports[monthKey]) {
        reports[monthKey] = {
          totalIncome: 0,
          totalExpense: 0,
          categoryBreakdown: {},
        };
      }

      const amount = expense.amount;
      const category = expense.category;

      if (expense.type === 'income') {
        reports[monthKey].totalIncome += amount;
      } else {
        reports[monthKey].totalExpense += amount;
      }

      // Track spending per category (Income is tracked separately)
      if (expense.type === 'expense') {
        if (!reports[monthKey].categoryBreakdown[category]) {
          reports[monthKey].categoryBreakdown[category] = 0;
        }
        reports[monthKey].categoryBreakdown[category] += amount;
      }
    });

    // Convert to an array and calculate net balance
    return Object.keys(reports).sort().reverse().map(monthKey => {
      const report = reports[monthKey];
      const netBalance = report.totalIncome - report.totalExpense;
      return {
        monthKey,
        totalIncome: report.totalIncome,
        totalExpense: report.totalExpense,
        netBalance,
        categoryBreakdown: report.categoryBreakdown,
      };
    });
  }, [expenses]);


  // --- Helper Components for Rendering ---

  /**
   * @param {{ report: { monthKey: string, totalIncome: number, totalExpense: number, netBalance: number, categoryBreakdown: {[category: string]: number} } }} props
   */
  const MonthlyReportCard = ({ report }) => {
    const totalExpense = report.totalExpense;
    // Format month key (YYYY-MM) to MMMM YYYY
    const [year, month] = report.monthKey.split('-');
    const date = new Date(year, month - 1);
    const monthName = date.toLocaleString('en-US', { month: 'long', year: 'numeric' });

    return (
      <div className="bg-white p-6 rounded-xl shadow-lg border border-gray-100 mb-4 transition duration-300 hover:shadow-xl">
        <h3 className="text-2xl font-bold mb-4 text-indigo-700">{monthName} Report</h3>

        <div className="grid grid-cols-3 gap-3 text-center mb-5">
          <div className="p-3 bg-green-50 rounded-lg">
            <p className="text-sm text-green-600">Income</p>
            <p className="text-lg font-semibold text-green-700">${report.totalIncome.toFixed(2)}</p>
          </div>
          <div className="p-3 bg-red-50 rounded-lg">
            <p className="text-sm text-red-600">Expenses</p>
            <p className="text-lg font-semibold text-red-700">${report.totalExpense.toFixed(2)}</p>
          </div>
          <div className={`p-3 rounded-lg ${report.netBalance >= 0 ? 'bg-indigo-50' : 'bg-yellow-50'}`}>
            <p className="text-sm text-indigo-600">Net Balance</p>
            <p className={`text-lg font-bold ${report.netBalance >= 0 ? 'text-indigo-700' : 'text-yellow-700'}`}>
              ${report.netBalance.toFixed(2)}
            </p>
          </div>
        </div>

        <h4 className="text-lg font-semibold mb-3 text-gray-700 border-b pb-2">Category Breakdown</h4>
        <div className="space-y-2 max-h-48 overflow-y-auto pr-2">
          {Object.entries(report.categoryBreakdown)
            .sort(([, a], [, b]) => b - a)
            .map(([category, amount]) => {
              const percentage = totalExpense > 0 ? ((amount / totalExpense) * 100).toFixed(1) : 0;
              const categoryInfo = CATEGORIES.find(c => c.name === category) || { icon: '❓' };
              return (
                <div key={category} className="flex justify-between items-center text-sm">
                  <span className="flex items-center text-gray-600">
                    {categoryInfo.icon} {category}
                  </span>
                  <div className="flex items-center space-x-2 w-2/3">
                    <div className="h-2 flex-grow bg-gray-200 rounded-full overflow-hidden">
                      <div
                        className="h-2 bg-red-500 rounded-full"
                        style={{ width: `${percentage}%` }}
                      ></div>
                    </div>
                    <span className="text-xs font-medium w-1/4 text-right">${amount.toFixed(2)} ({percentage}%)</span>
                  </div>
                </div>
              );
            })}
        </div>
      </div>
    );
  };

  /**
   * @param {{ expense: Expense }} props
   */
  const TransactionItem = ({ expense }) => {
    const categoryInfo = CATEGORIES.find(c => c.name === expense.category);
    const isIncome = expense.type === 'income';
    const amountClass = isIncome ? 'text-green-600' : 'text-red-600';
    const sign = isIncome ? '+' : '-';

    return (
      <li className="flex justify-between items-center p-3 bg-gray-50 rounded-lg hover:bg-gray-100 transition duration-150">
        <div className="flex items-center space-x-3">
          <span className="text-xl">{categoryInfo?.icon || '📦'}</span>
          <div>
            <p className="font-semibold text-gray-800">{expense.category}</p>
            <p className="text-xs text-gray-500 truncate max-w-xs">{expense.description}</p>
          </div>
        </div>
        <div className="text-right">
          <p className={`font-bold ${amountClass}`}>
            {sign} ${expense.amount.toFixed(2)}
          </p>
          <p className="text-xs text-gray-400">{expense.date}</p>
        </div>
      </li>
    );
  };


  // --- Main Render ---

  if (!isAuthReady) {
    return (
      <div className="min-h-screen flex items-center justify-center bg-gray-100">
        <div className="text-center p-8 bg-white rounded-xl shadow-lg">
          <div className="animate-spin rounded-full h-8 w-8 border-b-2 border-indigo-500 mx-auto mb-4"></div>
          <p className="text-gray-600 font-medium">Connecting to secure service...</p>
          {authError && <p className="mt-4 text-red-500 text-sm">Error: {authError}</p>}
          <p className="mt-2 text-xs text-gray-400">User ID: {userId || 'Authenticating...'}</p>
        </div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-gray-50 p-4 sm:p-8">
      <div className="max-w-7xl mx-auto">
        <h1 className="text-4xl font-extrabold text-indigo-800 mb-2">Personal Finance Tracker</h1>
        <p className="text-gray-500 mb-6">User ID: <span className="font-mono text-xs bg-gray-200 p-1 rounded">{userId}</span></p>

        <div className="grid grid-cols-1 lg:grid-cols-3 gap-8">
          {/* Column 1: Add New Transaction */}
          <div className="lg:col-span-1">
            <div className="bg-white p-6 rounded-xl shadow-2xl sticky top-4">
              <h2 className="text-2xl font-bold text-indigo-600 mb-4 border-b pb-2">Add New Transaction</h2>

              <form onSubmit={handleAddExpense} className="space-y-4">
                {/* Amount Field */}
                <div>
                  <label htmlFor="amount" className="block text-sm font-medium text-gray-700">Amount (USD)</label>
                  <input
                    type="number"
                    id="amount"
                    name="amount"
                    value={newExpense.amount}
                    onChange={handleInputChange}
                    placeholder="e.g., 50.50 (Positive amount only)"
                    required
                    step="0.01"
                    min="0.01"
                    className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 sm:text-sm"
                  />
                </div>

                {/* Category Field */}
                <div>
                  <label htmlFor="category" className="block text-sm font-medium text-gray-700">Category</label>
                  <select
                    id="category"
                    name="category"
                    value={newExpense.category}
                    onChange={handleInputChange}
                    required
                    className="mt-1 block w-full px-3 py-2 border border-gray-300 bg-white rounded-md shadow-sm focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 sm:text-sm"
                  >
                    {CATEGORIES.map(cat => (
                      <option key={cat.name} value={cat.name}>
                        {cat.name} ({cat.type.charAt(0).toUpperCase() + cat.type.slice(1)})
                      </option>
                    ))}
                  </select>
                </div>

                {/* Date Field */}
                <div>
                  <label htmlFor="date" className="block text-sm font-medium text-gray-700">Date</label>
                  <input
                    type="date"
                    id="date"
                    name="date"
                    value={newExpense.date}
                    onChange={handleInputChange}
                    required
                    className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 sm:text-sm"
                  />
                </div>

                {/* Description Field */}
                <div>
                  <label htmlFor="description" className="block text-sm font-medium text-gray-700">Description (Optional)</label>
                  <textarea
                    id="description"
                    name="description"
                    value={newExpense.description}
                    onChange={handleInputChange}
                    rows="2"
                    className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 sm:text-sm"
                  ></textarea>
                </div>

                {/* Submit Button */}
                <button
                  type="submit"
                  disabled={savingExpense}
                  className="w-full flex justify-center py-2 px-4 border border-transparent rounded-md shadow-sm text-sm font-medium text-white bg-indigo-600 hover:bg-indigo-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-indigo-500 disabled:opacity-50 transition duration-150"
                >
                  {savingExpense ? (
                    <div className="flex items-center">
                      <div className="animate-spin rounded-full h-5 w-5 border-b-2 border-white mr-2"></div>
                      Saving...
                    </div>
                  ) : (
                    `Add ${newExpense.type === 'income' ? 'Income' : 'Expense'}`
                  )}
                </button>
              </form>
            </div>
          </div>

          {/* Column 2 & 3: Reports and Transactions */}
          <div className="lg:col-span-2 space-y-8">
            {/* Reports Section */}
            <div className="bg-white p-6 rounded-xl shadow-2xl">
              <h2 className="text-2xl font-bold text-indigo-600 mb-4 border-b pb-2">Monthly Reports</h2>
              {loadingExpenses ? (
                <div className="text-center py-8">
                  <div className="animate-pulse text-indigo-500">Loading Report Data...</div>
                </div>
              ) : monthlyReports.length > 0 ? (
                <div className="space-y-4 max-h-96 overflow-y-auto pr-2">
                  {monthlyReports.map(report => (
                    <MonthlyReportCard key={report.monthKey} report={report} />
                  ))}
                </div>
              ) : (
                <p className="text-gray-500">No transactions recorded yet. Add one to see your first report!</p>
              )}
            </div>

            {/* Transactions Section */}
            <div className="bg-white p-6 rounded-xl shadow-2xl">
              <h2 className="text-2xl font-bold text-indigo-600 mb-4 border-b pb-2">Recent Transactions</h2>
              {loadingExpenses ? (
                <div className="text-center py-8">
                  <div className="animate-pulse text-indigo-500">Loading Transactions...</div>
                </div>
              ) : expenses.length > 0 ? (
                <ul className="space-y-3 max-h-96 overflow-y-auto pr-2">
                  {expenses.slice(0, 20).map(expense => (
                    <TransactionItem key={expense.id} expense={expense} />
                  ))}
                  {expenses.length > 20 && (
                    <li className="text-center text-sm text-gray-500 mt-4">Showing the 20 most recent transactions.</li>
                  )}
                </ul>
              ) : (
                <p className="text-gray-500">No transactions found. Start tracking your finances now!</p>
              )}
            </div>
          </div>
        </div>
      </div>
    </div>
  );
};

export default App;# Monthly-Expences-1
The Personal Finance Tracker uses React for the UI and Firebase Firestore for real-time, secure data storage. It allows users to add categorized transactions and generates up-to-date monthly reports showing net balance and spending breakdown.
