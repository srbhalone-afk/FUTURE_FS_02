# FUTURE_FS_02
.
Author - Saurabh Alone

import React, { useState, useEffect, useMemo } from 'react';
import { Header } from './components/Header';
import { LeadStats } from './components/LeadStats';
import { LeadFilterBar } from './components/LeadFilterBar';
import { LeadTable } from './components/LeadTable';
import { LeadKanban } from './components/LeadKanban';
import { LeadDetailsModal } from './components/LeadDetailsModal';
import { PublicContactForm } from './components/PublicContactForm';
import { AnalyticsView } from './components/AnalyticsView';
import { CreateLeadModal } from './components/CreateLeadModal';
import { AdminLoginModal } from './components/AdminLoginModal';
import {
  fetchLeads,
  fetchAnalytics,
  updateLeadStatus,
  addLeadNote,
  deleteLead,
  resetDemoData,
} from './utils/api';
import { Lead, LeadAnalytics, LeadStatus, AdminUser } from './types';
import { Sparkles, CheckCircle2, ShieldAlert } from 'lucide-react';

export default function App() {
  const [currentTab, setCurrentTab] = useState<'pipeline' | 'contact-form' | 'analytics'>('pipeline');
  const [leads, setLeads] = useState<Lead[]>([]);
  const [analytics, setAnalytics] = useState<LeadAnalytics | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  const [errorMsg, setErrorMsg] = useState<string | null>(null);

  // Filters & Search
  const [searchTerm, setSearchTerm] = useState('');
  const [selectedStatus, setSelectedStatus] = useState<string>('all');
  const [selectedSource, setSelectedSource] = useState<string>('all');
  const [sortBy, setSortBy] = useState<string>('newest');
  const [viewMode, setViewMode] = useState<'table' | 'kanban'>('table');

  // Selected lead & modals
  const [selectedLead, setSelectedLead] = useState<Lead | null>(null);
  const [isDetailsOpen, setIsDetailsOpen] = useState(false);
  const [isCreateModalOpen, setIsCreateModalOpen] = useState(false);
  const [isLoginModalOpen, setIsLoginModalOpen] = useState(false);

  // Admin authentication state (default signed in as admin for smooth initial experience, but allows logout/re-login)
  const [adminUser, setAdminUser] = useState<AdminUser | null>(() => {
    const saved = localStorage.getItem('crm_admin_user');
    if (saved) {
      try {
        return JSON.parse(saved);
      } catch {
        // fallback
      }
    }
    return {
      username: 'admin',
      name: 'CRM Administrator',
      role: 'Admin',
      token: 'token_initial',
    };
  });

  // Toast notification
  const [toastMessage, setToastMessage] = useState<string | null>(null);

  const showToast = (msg: string) => {
    setToastMessage(msg);
    setTimeout(() => {
      setToastMessage((current) => (current === msg ? null : current));
    }, 4000);
  };

  // Fetch leads and analytics from server
  const loadData = async () => {
    try {
      setIsLoading(true);
      setErrorMsg(null);
      const [leadsRes, analyticsRes] = await Promise.all([
        fetchLeads({
          status: selectedStatus,
          source: selectedSource,
          search: searchTerm,
          sort: sortBy,
        }),
        fetchAnalytics(),
      ]);

      setLeads(leadsRes.leads);
      setAnalytics(analyticsRes);
    } catch (err: any) {
      console.error('Error loading leads:', err);
      setErrorMsg('Failed to load CRM leads from server. Please check the backend connection.');
    } finally {
      setIsLoading(false);
    }
  };

  useEffect(() => {
    loadData();
  }, [selectedStatus, selectedSource, searchTerm, sortBy]);

  // Compute status counts for filter badges
  const statusCounts = useMemo(() => {
    return {
      all: analytics ? analytics.totalLeads : leads.length,
      new: analytics ? analytics.newLeads : leads.filter((l) => l.status === 'new').length,
      contacted: analytics
        ? analytics.contactedLeads
        : leads.filter((l) => l.status === 'contacted').length,
      converted: analytics
        ? analytics.convertedLeads
        : leads.filter((l) => l.status === 'converted').length,
      lost: analytics ? analytics.lostLeads : leads.filter((l) => l.status === 'lost').length,
    };
  }, [analytics, leads]);

  // Update status handler
  const handleUpdateStatus = async (leadId: string, status: LeadStatus, note?: string) => {
    try {
      const res = await updateLeadStatus(leadId, status, note);
      setLeads((prev) => prev.map((l) => (l.id === leadId ? res.lead : l)));

      if (selectedLead && selectedLead.id === leadId) {
        setSelectedLead(res.lead);
      }

      // Refresh analytics
      fetchAnalytics()
        .then(setAnalytics)
        .catch(() => {});

      showToast(`Lead status updated to ${status.toUpperCase()}`);
    } catch (err: any) {
      console.error('Failed to update status:', err);
      alert('Could not update status: ' + err.message);
    }
  };

  // Add follow-up note handler
  const handleAddNote = async (
    leadId: string,
    noteData: {
      content: string;
      contactMethod: 'email' | 'call' | 'meeting' | 'note';
      nextFollowUpDate?: string;
    },
  ) => {
    try {
      const res = await addLeadNote(leadId, {
        ...noteData,
        author: adminUser ? adminUser.name : 'Admin',
      });

      setLeads((prev) => prev.map((l) => (l.id === leadId ? res.lead : l)));
      if (selectedLead && selectedLead.id === leadId) {
        setSelectedLead(res.lead);
      }

      fetchAnalytics()
        .then(setAnalytics)
        .catch(() => {});

      showToast('Follow-up note logged successfully');
    } catch (err: any) {
      console.error('Failed to add note:', err);
      alert('Could not add note: ' + err.message);
    }
  };

  // Delete lead handler
  const handleDeleteLead = async (leadId: string) => {
    try {
      await deleteLead(leadId);
      setLeads((prev) => prev.filter((l) => l.id !== leadId));
      if (selectedLead && selectedLead.id === leadId) {
        setIsDetailsOpen(false);
        setSelectedLead(null);
      }

      fetchAnalytics()
        .then(setAnalytics)
        .catch(() => {});

      showToast('Lead deleted from database');
    } catch (err: any) {
      console.error('Failed to delete lead:', err);
      alert('Could not delete lead: ' + err.message);
    }
  };

  // Reset sample data
  const handleResetDemo = async () => {
    if (!window.confirm('Reset all leads to default demo data? Any custom inquiries will be replaced.')) {
      return;
    }
    try {
      await resetDemoData();
      await loadData();
      showToast('Sample CRM leads restored!');
    } catch (err: any) {
      alert('Failed to reset: ' + err.message);
    }
  };

  // Handle lead submitted from public contact form
  const handleLeadSubmitted = (newLead: Lead) => {
    setLeads((prev) => [newLead, ...prev]);
    fetchAnalytics()
      .then(setAnalytics)
      .catch(() => {});
    showToast(`New Lead received from ${newLead.name}!`);
  };

  // Auth handlers
  const handleLoginSuccess = (user: AdminUser) => {
    setAdminUser(user);
    localStorage.setItem('crm_admin_user', JSON.stringify(user));
    showToast(`Welcome back, ${user.name}`);
  };

  const handleLogout = () => {
    setAdminUser(null);
    localStorage.removeItem('crm_admin_user');
    showToast('Logged out of admin mode');
  };

  return (
    <div className="min-h-screen bg-stone-50/60 text-stone-900 flex flex-col font-sans selection:bg-amber-100 selection:text-amber-900">
      {/* Navigation Header */}
      <Header
        currentTab={currentTab}
        onSelectTab={setCurrentTab}
        adminUser={adminUser}
        onLogout={handleLogout}
        onOpenLogin={() => setIsLoginModalOpen(true)}
        onResetDemo={handleResetDemo}
        totalLeadsCount={statusCounts.all}
        newLeadsCount={statusCounts.new}
      />

      {/* Toast Banner */}
      {toastMessage && (
        <div className="fixed bottom-5 right-5 z-50 animate-in slide-in-from-bottom-5 duration-200">
          <div className="bg-stone-900 text-white px-4 py-2.5 rounded-xl shadow-xl flex items-center gap-2.5 text-xs font-medium border border-stone-800">
            <CheckCircle2 className="w-4 h-4 text-emerald-400 shrink-0" />
            <span>{toastMessage}</span>
          </div>
        </div>
      )}

      {/* Main Content Area */}
      <main className="flex-1 max-w-7xl w-full mx-auto px-4 sm:px-6 lg:px-8 py-6 sm:py-8">
        {/* TAB 1: PIPELINE / LEADS LIST */}
        {currentTab === 'pipeline' && (
          <div>
            {/* Non-admin notice if logged out */}
            {!adminUser && (
              <div className="mb-6 p-4 rounded-xl bg-amber-50 border border-amber-200 flex items-center justify-between gap-4">
                <div className="flex items-center gap-2.5 text-xs text-amber-900">
                  <ShieldAlert className="w-4 h-4 text-amber-600 shrink-0" />
                  <span>
                    You are currently viewing in guest mode. Sign in to admin mode for full control and editing privileges.
                  </span>
                </div>
                <button
                  onClick={() => setIsLoginModalOpen(true)}
                  className="px-3 py-1 bg-amber-600 hover:bg-amber-700 text-white text-xs font-semibold rounded-lg shrink-0 transition-colors"
                >
                  Admin Sign In
                </button>
              </div>
            )}

            {/* KPI Metric Cards */}
            <LeadStats
              totalCount={statusCounts.all}
              newCount={statusCounts.new}
              contactedCount={statusCounts.contacted}
              convertedCount={statusCounts.converted}
              conversionRate={analytics?.conversionRate || 0}
              selectedStatusFilter={selectedStatus}
              onSelectStatus={setSelectedStatus}
            />

            {/* Filter, Search, and Action Bar */}
            <LeadFilterBar
              searchTerm={searchTerm}
              onSearchChange={setSearchTerm}
              selectedStatus={selectedStatus}
              onStatusChange={setSelectedStatus}
              selectedSource={selectedSource}
              onSourceChange={setSelectedSource}
              sortBy={sortBy}
              onSortChange={setSortBy}
              viewMode={viewMode}
              onViewModeChange={setViewMode}
              onOpenCreateModal={() => setIsCreateModalOpen(true)}
              statusCounts={statusCounts}
            />

            {/* Leads View: Table or Kanban */}
            {isLoading ? (
              <div className="bg-white rounded-xl border border-stone-200 p-12 text-center text-stone-400 text-sm">
                Loading lead records...
              </div>
            ) : viewMode === 'table' ? (
              <LeadTable
                leads={leads}
                onSelectLead={(lead) => {
                  setSelectedLead(lead);
                  setIsDetailsOpen(true);
                }}
                onUpdateStatus={handleUpdateStatus}
                onDeleteLead={handleDeleteLead}
              />
            ) : (
              <LeadKanban
                leads={leads}
                onSelectLead={(lead) => {
                  setSelectedLead(lead);
                  setIsDetailsOpen(true);
                }}
                onUpdateStatus={handleUpdateStatus}
              />
            )}
          </div>
        )}

        {/* TAB 2: PUBLIC CONTACT FORM */}
        {currentTab === 'contact-form' && (
          <PublicContactForm
            onLeadSubmitted={handleLeadSubmitted}
            onGoToPipeline={() => setCurrentTab('pipeline')}
          />
        )}

        {/* TAB 3: ANALYTICS & INSIGHTS */}
        {currentTab === 'analytics' && (
          <AnalyticsView
            analytics={analytics}
            leads={leads}
            onSelectLead={(lead) => {
              setSelectedLead(lead);
              setIsDetailsOpen(true);
            }}
          />
        )}
      </main>

      {/* Lead Details & Follow-up Notes Modal */}
      <LeadDetailsModal
        lead={selectedLead}
        isOpen={isDetailsOpen}
        onClose={() => {
          setIsDetailsOpen(false);
          setSelectedLead(null);
        }}
        onUpdateStatus={handleUpdateStatus}
        onAddNote={handleAddNote}
      />

      {/* Manual Create Lead Modal */}
      <CreateLeadModal
        isOpen={isCreateModalOpen}
        onClose={() => setIsCreateModalOpen(false)}
        onLeadCreated={(lead) => {
          handleLeadSubmitted(lead);
          showToast(`Lead for ${lead.name} logged`);
        }}
      />

      {/* Admin Login Modal */}
      <AdminLoginModal
        isOpen={isLoginModalOpen}
        onClose={() => setIsLoginModalOpen(false)}
        onLoginSuccess={handleLoginSuccess}
      />
    </div>
  );
}
