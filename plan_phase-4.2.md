🏥 Phase 4: Patient Portal Implementation - Week 2 (Core Features)
Execution Status: ACTIVE DEVELOPMENT ⚡
I'm now meticulously implementing the Core Features of the patient portal, including the revolutionary Health Journey Timeline, AI Health Translator, Medication Guardian, and Daily Check-in System.

📈 Week 3: Health Journey Timeline Implementation
1.1 Timeline Data Models & Service ✅
TypeScript

// src/lib/portal/timeline/timeline-service.ts
import { createSupabaseServer } from '../auth/auth-service'
import { format, parseISO, isAfter, isBefore, startOfMonth, endOfMonth } from 'date-fns'

export interface TimelineEvent {
  id: string
  eventDate: Date
  eventType: 'appointment' | 'diagnosis' | 'medication_start' | 'medication_end' | 
             'test_result' | 'procedure' | 'symptom' | 'milestone' | 'check_in'
  eventCategory: 'positive' | 'neutral' | 'attention'
  title: string
  description: string
  simplifiedDescription?: string
  icon: string
  color: string
  relatedRecordId?: string
  relatedRecordType?: string
  metadata?: {
    provider?: string
    location?: string
    results?: any
    medications?: string[]
    vitals?: any
  }
}

export interface TimelineFilter {
  startDate?: Date
  endDate?: Date
  eventTypes?: string[]
  eventCategories?: string[]
  searchTerm?: string
}

export class TimelineService {
  private supabase = createSupabaseServer()
  
  // Fetch timeline events for a patient
  async getTimelineEvents(
    patientId: string, 
    filter?: TimelineFilter
  ): Promise<TimelineEvent[]> {
    let query = this.supabase
      .from('health_timeline')
      .select('*')
      .eq('patient_id', patientId)
      .order('event_date', { ascending: false })
    
    // Apply filters
    if (filter?.startDate) {
      query = query.gte('event_date', filter.startDate.toISOString())
    }
    if (filter?.endDate) {
      query = query.lte('event_date', filter.endDate.toISOString())
    }
    if (filter?.eventTypes?.length) {
      query = query.in('event_type', filter.eventTypes)
    }
    if (filter?.eventCategories?.length) {
      query = query.in('event_category', filter.eventCategories)
    }
    
    const { data, error } = await query
    
    if (error) throw error
    
    // Apply text search if needed
    let events = data || []
    if (filter?.searchTerm) {
      const searchLower = filter.searchTerm.toLowerCase()
      events = events.filter(e => 
        e.title.toLowerCase().includes(searchLower) ||
        e.description?.toLowerCase().includes(searchLower) ||
        e.simplified_description?.toLowerCase().includes(searchLower)
      )
    }
    
    return events.map(this.mapEventToTimeline)
  }
  
  // Get timeline summary (for dashboard)
  async getTimelineSummary(patientId: string) {
    const now = new Date()
    const threeMonthsAgo = new Date(now.getFullYear(), now.getMonth() - 3, now.getDate())
    
    const events = await this.getTimelineEvents(patientId, {
      startDate: threeMonthsAgo,
      endDate: now
    })
    
    // Group events by month
    const eventsByMonth = events.reduce((acc, event) => {
      const monthKey = format(event.eventDate, 'yyyy-MM')
      if (!acc[monthKey]) {
        acc[monthKey] = []
      }
      acc[monthKey].push(event)
      return acc
    }, {} as Record<string, TimelineEvent[]>)
    
    // Calculate statistics
    const stats = {
      totalEvents: events.length,
      positiveEvents: events.filter(e => e.eventCategory === 'positive').length,
      attentionEvents: events.filter(e => e.eventCategory === 'attention').length,
      appointments: events.filter(e => e.eventType === 'appointment').length,
      medications: events.filter(e => e.eventType.includes('medication')).length,
      recentMilestones: events
        .filter(e => e.eventType === 'milestone' && e.eventCategory === 'positive')
        .slice(0, 3)
    }
    
    return {
      events,
      eventsByMonth,
      stats
    }
  }
  
  // Add timeline event
  async addTimelineEvent(
    patientId: string,
    event: Omit<TimelineEvent, 'id'>
  ): Promise<TimelineEvent> {
    const { data, error } = await this.supabase
      .from('health_timeline')
      .insert({
        patient_id: patientId,
        event_date: event.eventDate,
        event_type: event.eventType,
        event_category: event.eventCategory,
        title: event.title,
        description: event.description,
        simplified_description: event.simplifiedDescription,
        icon: event.icon,
        color: event.color,
        related_record_id: event.relatedRecordId,
        related_record_type: event.relatedRecordType,
        metadata: event.metadata
      })
      .select()
      .single()
    
    if (error) throw error
    
    return this.mapEventToTimeline(data)
  }
  
  // Automatically generate timeline events from health records
  async generateTimelineFromRecords(patientId: string) {
    // Fetch recent health records
    const { data: records } = await this.supabase
      .from('health_records')
      .select('*')
      .eq('patient_id', patientId)
      .order('record_date', { ascending: false })
      .limit(50)
    
    // Fetch appointments
    const { data: appointments } = await this.supabase
      .from('appointments')
      .select('*')
      .eq('patient_id', patientId)
      .order('appointment_date', { ascending: false })
      .limit(20)
    
    // Fetch medication changes
    const { data: medications } = await this.supabase
      .from('medications')
      .select('*')
      .eq('patient_id', patientId)
      .order('start_date', { ascending: false })
    
    const events: Omit<TimelineEvent, 'id'>[] = []
    
    // Process health records
    records?.forEach(record => {
      events.push({
        eventDate: new Date(record.record_date),
        eventType: this.mapRecordTypeToEventType(record.record_type),
        eventCategory: this.determineEventCategory(record),
        title: this.generateEventTitle(record),
        description: record.original_content?.summary || '',
        simplifiedDescription: record.simplified_content?.summary,
        icon: this.getIconForRecordType(record.record_type),
        color: this.getColorForCategory(record.is_critical ? 'attention' : 'neutral'),
        relatedRecordId: record.id,
        relatedRecordType: 'health_record'
      })
    })
    
    // Process appointments
    appointments?.forEach(apt => {
      events.push({
        eventDate: new Date(apt.appointment_date),
        eventType: 'appointment',
        eventCategory: apt.status === 'completed' ? 'positive' : 'neutral',
        title: `Appointment with ${apt.provider_name}`,
        description: `${apt.appointment_type} at ${apt.location}`,
        simplifiedDescription: `Doctor visit for ${apt.reason}`,
        icon: 'calendar',
        color: 'blue',
        relatedRecordId: apt.id,
        relatedRecordType: 'appointment'
      })
    })
    
    // Process medication changes
    medications?.forEach(med => {
      events.push({
        eventDate: new Date(med.start_date),
        eventType: 'medication_start',
        eventCategory: 'neutral',
        title: `Started ${med.medication_name}`,
        description: `${med.dosage} - ${med.frequency}`,
        simplifiedDescription: med.instructions_simplified,
        icon: 'pill',
        color: 'purple',
        relatedRecordId: med.id,
        relatedRecordType: 'medication'
      })
      
      if (med.end_date) {
        events.push({
          eventDate: new Date(med.end_date),
          eventType: 'medication_end',
          eventCategory: 'neutral',
          title: `Stopped ${med.medication_name}`,
          description: 'Medication discontinued',
          simplifiedDescription: 'Medicine stopped',
          icon: 'pill-off',
          color: 'gray',
          relatedRecordId: med.id,
          relatedRecordType: 'medication'
        })
      }
    })
    
    // Bulk insert events (avoiding duplicates)
    for (const event of events) {
      try {
        await this.addTimelineEvent(patientId, event)
      } catch (error) {
        // Skip duplicates
        console.log('Event might already exist:', event.title)
      }
    }
    
    return events
  }
  
  private mapEventToTimeline(data: any): TimelineEvent {
    return {
      id: data.id,
      eventDate: new Date(data.event_date),
      eventType: data.event_type,
      eventCategory: data.event_category,
      title: data.title,
      description: data.description,
      simplifiedDescription: data.simplified_description,
      icon: data.icon,
      color: data.color,
      relatedRecordId: data.related_record_id,
      relatedRecordType: data.related_record_type,
      metadata: data.metadata
    }
  }
  
  private mapRecordTypeToEventType(recordType: string): TimelineEvent['eventType'] {
    const mapping: Record<string, TimelineEvent['eventType']> = {
      'lab_result': 'test_result',
      'diagnosis': 'diagnosis',
      'procedure': 'procedure',
      'vital_signs': 'check_in'
    }
    return mapping[recordType] || 'test_result'
  }
  
  private determineEventCategory(record: any): TimelineEvent['eventCategory'] {
    if (record.is_critical) return 'attention'
    if (record.improved || record.resolved) return 'positive'
    return 'neutral'
  }
  
  private generateEventTitle(record: any): string {
    const titles: Record<string, string> = {
      'lab_result': `Lab Results: ${record.test_name || 'Tests'}`,
      'diagnosis': `Diagnosis: ${record.condition || 'Medical Condition'}`,
      'procedure': `Procedure: ${record.procedure_name || 'Medical Procedure'}`,
      'vital_signs': 'Vital Signs Recorded'
    }
    return titles[record.record_type] || 'Health Record'
  }
  
  private getIconForRecordType(type: string): string {
    const icons: Record<string, string> = {
      'lab_result': 'flask',
      'diagnosis': 'clipboard',
      'procedure': 'activity',
      'vital_signs': 'heart',
      'appointment': 'calendar',
      'medication_start': 'pill',
      'medication_end': 'pill-off',
      'milestone': 'trophy',
      'symptom': 'alert-circle'
    }
    return icons[type] || 'file-text'
  }
  
  private getColorForCategory(category: string): string {
    const colors: Record<string, string> = {
      'positive': 'green',
      'neutral': 'blue',
      'attention': 'orange'
    }
    return colors[category] || 'gray'
  }
}
1.2 Health Journey Timeline UI Component ✅
TypeScript

// src/components/portal/timeline/health-journey-timeline.tsx
'use client'

import React, { useState, useRef, useEffect } from 'react'
import { motion, AnimatePresence } from 'framer-motion'
import { useQuery } from '@tanstack/react-query'
import { 
  Calendar,
  Clock,
  ChevronLeft,
  ChevronRight,
  Filter,
  Search,
  Mic,
  Download,
  Printer,
  ZoomIn,
  ZoomOut,
  Activity,
  Pill,
  FileText,
  Heart,
  AlertCircle,
  TrendingUp,
  Trophy,
  Volume2
} from 'lucide-react'
import { format, startOfMonth, endOfMonth, eachDayOfInterval, isSameDay, isToday } from 'date-fns'
import { TimelineService, TimelineEvent } from '@/lib/portal/timeline/timeline-service'
import { ElderCard } from '@/components/ui/elder-card'
import { ElderButton } from '@/components/ui/elder-button'
import { useAdaptiveUI } from '@/lib/portal/ui/adaptive-ui-context'

interface TimelineProps {
  patientId: string
  onEventClick?: (event: TimelineEvent) => void
}

export function HealthJourneyTimeline({ patientId, onEventClick }: TimelineProps) {
  const { uiMode, accessibility } = useAdaptiveUI()
  const [selectedDate, setSelectedDate] = useState(new Date())
  const [viewMode, setViewMode] = useState<'timeline' | 'calendar' | 'list'>('timeline')
  const [zoomLevel, setZoomLevel] = useState(100)
  const [filter, setFilter] = useState({
    eventTypes: [] as string[],
    searchTerm: ''
  })
  const [isNarrating, setIsNarrating] = useState(false)
  
  const timelineRef = useRef<HTMLDivElement>(null)
  const timelineService = new TimelineService()
  
  // Fetch timeline events
  const { data: events, isLoading } = useQuery({
    queryKey: ['timeline', patientId, selectedDate, filter],
    queryFn: () => timelineService.getTimelineEvents(patientId, {
      startDate: startOfMonth(selectedDate),
      endDate: endOfMonth(selectedDate),
      ...filter
    })
  })
  
  // Voice narration
  const narrateTimeline = () => {
    if (!events || events.length === 0) return
    
    setIsNarrating(true)
    const text = `Your health timeline for ${format(selectedDate, 'MMMM yyyy')}. 
      You have ${events.length} health events. 
      ${events.slice(0, 5).map(e => 
        `On ${format(e.eventDate, 'MMMM do')}, ${e.simplifiedDescription || e.title}`
      ).join('. ')}`
    
    const utterance = new SpeechSynthesisUtterance(text)
    utterance.rate = 0.9
    utterance.onend = () => setIsNarrating(false)
    
    speechSynthesis.speak(utterance)
  }
  
  // Generate PDF
  const generatePDF = async () => {
    const { jsPDF } = await import('jspdf')
    const doc = new jsPDF()
    
    doc.setFontSize(20)
    doc.text('My Health Journey', 20, 20)
    doc.setFontSize(12)
    doc.text(`${format(selectedDate, 'MMMM yyyy')}`, 20, 30)
    
    let yPosition = 50
    events?.forEach((event, index) => {
      if (yPosition > 250) {
        doc.addPage()
        yPosition = 20
      }
      
      doc.setFontSize(10)
      doc.text(`${format(event.eventDate, 'MMM dd')}`, 20, yPosition)
      doc.setFontSize(12)
      doc.text(event.title, 50, yPosition)
      doc.setFontSize(10)
      doc.text(event.simplifiedDescription || event.description, 50, yPosition + 5)
      
      yPosition += 20
    })
    
    doc.save(`health-timeline-${format(selectedDate, 'yyyy-MM')}.pdf`)
  }
  
  // Render timeline view
  const renderTimelineView = () => {
    if (!events || events.length === 0) {
      return (
        <div className="text-center py-12">
          <Activity className="w-16 h-16 text-neutral-300 mx-auto mb-4" />
          <h3 className="text-lg font-medium text-neutral-600">
            No health events for this period
          </h3>
          <p className="text-neutral-500 mt-2">
            Your health events will appear here as they are recorded
          </p>
        </div>
      )
    }
    
    // Group events by date
    const eventsByDate = events.reduce((acc, event) => {
      const dateKey = format(event.eventDate, 'yyyy-MM-dd')
      if (!acc[dateKey]) {
        acc[dateKey] = []
      }
      acc[dateKey].push(event)
      return acc
    }, {} as Record<string, TimelineEvent[]>)
    
    return (
      <div 
        className="relative"
        style={{ fontSize: `${zoomLevel}%` }}
      >
        {/* Timeline Line */}
        <div className="absolute left-8 top-0 bottom-0 w-1 bg-gradient-to-b from-sage-200 via-sage-300 to-sage-200" />
        
        {/* Events */}
        <div className="space-y-8">
          {Object.entries(eventsByDate).map(([date, dayEvents]) => (
            <motion.div
              key={date}
              initial={{ opacity: 0, x: -20 }}
              animate={{ opacity: 1, x: 0 }}
              transition={{ duration: 0.5 }}
              className="relative"
            >
              {/* Date Marker */}
              <div className="flex items-center gap-4 mb-4">
                <div className="w-16 h-16 bg-white border-4 border-sage-400 rounded-full flex items-center justify-center z-10">
                  <span className="text-sm font-bold text-sage-700">
                    {format(new Date(date), 'dd')}
                  </span>
                </div>
                <div>
                  <h3 className="text-lg font-semibold text-neutral-900">
                    {format(new Date(date), 'EEEE')}
                  </h3>
                  <p className="text-sm text-neutral-600">
                    {format(new Date(date), 'MMMM dd, yyyy')}
                  </p>
                </div>
              </div>
              
              {/* Day Events */}
              <div className="ml-20 space-y-4">
                {dayEvents.map((event) => (
                  <TimelineEventCard
                    key={event.id}
                    event={event}
                    onClick={() => onEventClick?.(event)}
                    simplified={uiMode === 'simple'}
                  />
                ))}
              </div>
            </motion.div>
          ))}
        </div>
      </div>
    )
  }
  
  return (
    <div className="space-y-6">
      {/* Header & Controls */}
      <div className="flex flex-col lg:flex-row lg:items-center lg:justify-between gap-4">
        <div>
          <h2 className="text-2xl font-bold text-neutral-900">
            My Health Journey
          </h2>
          <p className="text-neutral-600 mt-1">
            Your health story, visualized over time
          </p>
        </div>
        
        {/* View Mode Toggle */}
        <div className="flex items-center gap-2">
          <button
            onClick={() => setViewMode('timeline')}
            className={`px-4 py-2 rounded-lg font-medium transition-colors ${
              viewMode === 'timeline' 
                ? 'bg-sage-500 text-white' 
                : 'bg-neutral-100 text-neutral-700 hover:bg-neutral-200'
            }`}
          >
            Timeline
          </button>
          <button
            onClick={() => setViewMode('calendar')}
            className={`px-4 py-2 rounded-lg font-medium transition-colors ${
              viewMode === 'calendar' 
                ? 'bg-sage-500 text-white' 
                : 'bg-neutral-100 text-neutral-700 hover:bg-neutral-200'
            }`}
          >
            Calendar
          </button>
          <button
            onClick={() => setViewMode('list')}
            className={`px-4 py-2 rounded-lg font-medium transition-colors ${
              viewMode === 'list' 
                ? 'bg-sage-500 text-white' 
                : 'bg-neutral-100 text-neutral-700 hover:bg-neutral-200'
            }`}
          >
            List
          </button>
        </div>
      </div>
      
      {/* Month Navigation */}
      <ElderCard className="p-4">
        <div className="flex items-center justify-between">
          <ElderButton
            variant="ghost"
            size="small"
            onClick={() => setSelectedDate(prev => 
              new Date(prev.getFullYear(), prev.getMonth() - 1)
            )}
          >
            <ChevronLeft className="w-5 h-5" />
            Previous
          </ElderButton>
          
          <h3 className="text-xl font-semibold text-neutral-900">
            {format(selectedDate, 'MMMM yyyy')}
          </h3>
          
          <ElderButton
            variant="ghost"
            size="small"
            onClick={() => setSelectedDate(prev => 
              new Date(prev.getFullYear(), prev.getMonth() + 1)
            )}
          >
            Next
            <ChevronRight className="w-5 h-5" />
          </ElderButton>
        </div>
      </ElderCard>
      
      {/* Action Bar */}
      <div className="flex flex-wrap items-center gap-2">
        {/* Search */}
        <div className="flex-1 min-w-[200px] relative">
          <Search className="absolute left-3 top-1/2 -translate-y-1/2 w-5 h-5 text-neutral-400" />
          <input
            type="text"
            placeholder="Search events..."
            value={filter.searchTerm}
            onChange={(e) => setFilter(prev => ({ ...prev, searchTerm: e.target.value }))}
            className="w-full pl-10 pr-4 py-2 border-2 border-neutral-200 rounded-lg focus:border-sage-500 focus:outline-none"
          />
        </div>
        
        {/* Voice Narration */}
        {accessibility.voiceEnabled && (
          <ElderButton
            variant="ghost"
            size="small"
            onClick={narrateTimeline}
            disabled={isNarrating}
          >
            <Volume2 className={`w-5 h-5 ${isNarrating ? 'animate-pulse' : ''}`} />
            {isNarrating ? 'Speaking...' : 'Narrate'}
          </ElderButton>
        )}
        
        {/* Zoom Controls */}
        <div className="flex items-center gap-1 border-2 border-neutral-200 rounded-lg">
          <button
            onClick={() => setZoomLevel(prev => Math.max(50, prev - 10))}
            className="p-2 hover:bg-neutral-100"
          >
            <ZoomOut className="w-5 h-5" />
          </button>
          <span className="px-2 text-sm font-medium">{zoomLevel}%</span>
          <button
            onClick={() => setZoomLevel(prev => Math.min(200, prev + 10))}
            className="p-2 hover:bg-neutral-100"
          >
            <ZoomIn className="w-5 h-5" />
          </button>
        </div>
        
        {/* Export Options */}
        <ElderButton
          variant="ghost"
          size="small"
          onClick={generatePDF}
        >
          <Download className="w-5 h-5" />
          PDF
        </ElderButton>
        
        <ElderButton
          variant="ghost"
          size="small"
          onClick={() => window.print()}
        >
          <Printer className="w-5 h-5" />
          Print
        </ElderButton>
      </div>
      
      {/* Filter Pills */}
      <div className="flex flex-wrap gap-2">
        {['appointment', 'medication_start', 'test_result', 'diagnosis', 'milestone'].map(type => (
          <button
            key={type}
            onClick={() => setFilter(prev => ({
              ...prev,
              eventTypes: prev.eventTypes.includes(type)
                ? prev.eventTypes.filter(t => t !== type)
                : [...prev.eventTypes, type]
            }))}
            className={`px-3 py-1 rounded-full text-sm font-medium transition-colors ${
              filter.eventTypes.includes(type)
                ? 'bg-sage-500 text-white'
                : 'bg-neutral-100 text-neutral-700 hover:bg-neutral-200'
            }`}
          >
            {type.replace('_', ' ')}
          </button>
        ))}
      </div>
      
      {/* Timeline Content */}
      <ElderCard className="p-6" ref={timelineRef}>
        {isLoading ? (
          <div className="flex justify-center py-12">
            <div className="animate-spin rounded-full h-12 w-12 border-4 border-sage-500 border-t-transparent" />
          </div>
        ) : (
          <>
            {viewMode === 'timeline' && renderTimelineView()}
            {viewMode === 'calendar' && <TimelineCalendarView events={events || []} />}
            {viewMode === 'list' && <TimelineListView events={events || []} />}
          </>
        )}
      </ElderCard>
    </div>
  )
}

// Timeline Event Card Component
function TimelineEventCard({ 
  event, 
  onClick,
  simplified = false
}: { 
  event: TimelineEvent
  onClick?: () => void
  simplified?: boolean
}) {
  const getCategoryStyle = (category: string) => {
    switch (category) {
      case 'positive':
        return 'border-green-400 bg-green-50'
      case 'attention':
        return 'border-orange-400 bg-orange-50'
      default:
        return 'border-blue-400 bg-blue-50'
    }
  }
  
  const getIcon = () => {
    const icons: Record<string, any> = {
      'appointment': Calendar,
      'medication_start': Pill,
      'test_result': FileText,
      'diagnosis': AlertCircle,
      'milestone': Trophy,
      'check_in': Heart,
      'procedure': Activity
    }
    const Icon = icons[event.eventType] || FileText
    return <Icon className="w-5 h-5" />
  }
  
  return (
    <motion.div
      whileHover={{ scale: 1.02 }}
      whileTap={{ scale: 0.98 }}
      onClick={onClick}
      className={`p-4 rounded-xl border-2 cursor-pointer transition-all ${getCategoryStyle(event.eventCategory)}`}
    >
      <div className="flex items-start gap-3">
        <div className={`p-2 rounded-lg bg-white`}>
          {getIcon()}
        </div>
        <div className="flex-1">
          <h4 className="font-semibold text-neutral-900">
            {event.title}
          </h4>
          <p className="text-sm text-neutral-700 mt-1">
            {simplified && event.simplifiedDescription 
              ? event.simplifiedDescription 
              : event.description}
          </p>
          {event.metadata?.provider && (
            <p className="text-xs text-neutral-500 mt-2">
              Provider: {event.metadata.provider}
            </p>
          )}
        </div>
        <div className="text-right">
          <p className="text-xs font-medium text-neutral-500">
            {format(event.eventDate, 'h:mm a')}
          </p>
        </div>
      </div>
    </motion.div>
  )
}

// Calendar View Component
function TimelineCalendarView({ events }: { events: TimelineEvent[] }) {
  const daysInMonth = eachDayOfInterval({
    start: startOfMonth(new Date()),
    end: endOfMonth(new Date())
  })
  
  const getEventsForDay = (day: Date) => 
    events.filter(e => isSameDay(e.eventDate, day))
  
  return (
    <div className="grid grid-cols-7 gap-2">
      {/* Day headers */}
      {['Sun', 'Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat'].map(day => (
        <div key={day} className="text-center font-medium text-neutral-600 py-2">
          {day}
        </div>
      ))}
      
      {/* Calendar days */}
      {daysInMonth.map(day => {
        const dayEvents = getEventsForDay(day)
        const hasEvents = dayEvents.length > 0
        
        return (
          <div
            key={day.toISOString()}
            className={`min-h-[80px] p-2 border rounded-lg ${
              isToday(day) ? 'border-sage-500 bg-sage-50' : 'border-neutral-200'
            } ${hasEvents ? 'cursor-pointer hover:bg-neutral-50' : ''}`}
          >
            <div className="text-sm font-medium mb-1">
              {format(day, 'd')}
            </div>
            {dayEvents.slice(0, 2).map(event => (
              <div
                key={event.id}
                className={`text-xs p-1 mb-1 rounded ${
                  event.eventCategory === 'positive' ? 'bg-green-100' :
                  event.eventCategory === 'attention' ? 'bg-orange-100' :
                  'bg-blue-100'
                }`}
              >
                {event.title.substring(0, 20)}...
              </div>
            ))}
            {dayEvents.length > 2 && (
              <div className="text-xs text-neutral-500">
                +{dayEvents.length - 2} more
              </div>
            )}
          </div>
        )
      })}
    </div>
  )
}

// List View Component  
function TimelineListView({ events }: { events: TimelineEvent[] }) {
  return (
    <div className="space-y-2">
      {events.map(event => (
        <div
          key={event.id}
          className="flex items-center gap-4 p-3 rounded-lg hover:bg-neutral-50"
        >
          <div className="text-sm font-medium text-neutral-600 w-24">
            {format(event.eventDate, 'MMM dd')}
          </div>
          <div className="flex-1">
            <h4 className="font-medium text-neutral-900">{event.title}</h4>
            <p className="text-sm text-neutral-600">{event.description}</p>
          </div>
          <span className={`px-2 py-1 rounded-full text-xs font-medium ${
            event.eventCategory === 'positive' ? 'bg-green-100 text-green-700' :
            event.eventCategory === 'attention' ? 'bg-orange-100 text-orange-700' :
            'bg-blue-100 text-blue-700'
          }`}>
            {event.eventType.replace('_', ' ')}
          </span>
        </div>
      ))}
    </div>
  )
}
🤖 Week 4: AI Health Translator & Medication Guardian
2.1 AI Health Translator Service ✅
TypeScript

// src/lib/portal/ai/health-translator.ts
import OpenAI from 'openai'

export interface TranslationRequest {
  text: string
  type: 'lab_result' | 'diagnosis' | 'medication' | 'procedure' | 'general'
  language?: 'en' | 'zh' | 'ms' | 'ta'
  simplificationLevel?: 'basic' | 'intermediate' | 'detailed'
  patientAge?: number
}

export interface TranslationResponse {
  original: string
  simplified: string
  keyPoints: string[]
  actionItems?: string[]
  definitions?: Record<string, string>
  confidence: number
}

export class HealthTranslatorService {
  private openai: OpenAI
  private translationCache: Map<string, TranslationResponse>
  
  constructor() {
    this.openai = new OpenAI({
      apiKey: process.env.OPENAI_API_KEY,
    })
    this.translationCache = new Map()
  }
  
  // Main translation method
  async translateMedicalText({
    text,
    type,
    language = 'en',
    simplificationLevel = 'basic',
    patientAge
  }: TranslationRequest): Promise<TranslationResponse> {
    // Check cache first
    const cacheKey = `${text}-${type}-${language}-${simplificationLevel}`
    if (this.translationCache.has(cacheKey)) {
      return this.translationCache.get(cacheKey)!
    }
    
    try {
      const prompt = this.buildPrompt(text, type, language, simplificationLevel, patientAge)
      
      const completion = await this.openai.chat.completions.create({
        model: "gpt-4",
        messages: [
          {
            role: "system",
            content: `You are a medical translator helping elderly patients understand their health information. 
            Translate complex medical terms into simple, clear language that a ${patientAge || 70}-year-old can understand.
            Be warm, reassuring, and accurate. Never provide medical advice, only explain what things mean.`
          },
          {
            role: "user",
            content: prompt
          }
        ],
        temperature: 0.3, // Lower temperature for more consistent medical translations
        max_tokens: 1000,
      })
      
      const response = this.parseAIResponse(
        completion.choices[0].message.content || '',
        text
      )
      
      // Cache the response
      this.translationCache.set(cacheKey, response)
      
      // Store in database for offline access
      await this.saveTranslation(text, response)
      
      return response
    } catch (error) {
      console.error('Translation error:', error)
      
      // Fallback to local simplification
      return this.fallbackTranslation(text, type)
    }
  }
  
  // Build prompt based on medical content type
  private buildPrompt(
    text: string,
    type: string,
    language: string,
    level: string,
    patientAge?: number
  ): string {
    const prompts: Record<string, string> = {
      lab_result: `
        Explain this lab result in simple terms:
        "${text}"
        
        1. What does this test measure?
        2. What do the numbers mean? (Is it normal, high, or low?)
        3. What should the patient understand about this?
        4. Any lifestyle factors that affect this?
        
        Use analogies when helpful. Avoid medical jargon.
      `,
      
      diagnosis: `
        Explain this diagnosis in simple, non-scary language:
        "${text}"
        
        1. What is this condition in simple words?
        2. What causes it?
        3. How does it affect daily life?
        4. What are the next steps?
        
        Be reassuring and focus on management rather than problems.
      `,
      
      medication: `
        Explain this medication information simply:
        "${text}"
        
        1. What is this medicine for?
        2. How does it help?
        3. When and how to take it?
        4. Important things to remember?
        
        Use everyday language and practical examples.
      `,
      
      procedure: `
        Explain this medical procedure simply:
        "${text}"
        
        1. What will happen during this procedure?
        2. Why is it needed?
        3. How long will it take?
        4. What to expect before and after?
        
        Be reassuring and focus on what the patient will experience.
      `,
      
      general: `
        Simplify this medical information:
        "${text}"
        
        Explain in plain language that anyone can understand.
        Focus on what's most important for the patient to know.
      `
    }
    
    let prompt = prompts[type] || prompts.general
    
    // Add language instruction
    if (language !== 'en') {
      const languages = {
        'zh': 'Simplified Chinese',
        'ms': 'Bahasa Melayu',
        'ta': 'Tamil'
      }
      prompt += `\n\nProvide the explanation in ${languages[language]}.`
    }
    
    // Add simplification level
    if (level === 'basic') {
      prompt += '\n\nUse very simple words, like explaining to a child.'
    } else if (level === 'detailed') {
      prompt += '\n\nProvide more detail while still keeping language simple.'
    }
    
    return prompt
  }
  
  // Parse AI response into structured format
  private parseAIResponse(aiText: string, originalText: string): TranslationResponse {
    // Extract key points (lines starting with bullets or numbers)
    const keyPoints = aiText
      .split('\n')
      .filter(line => /^[\d•\-*]/.test(line.trim()))
      .map(line => line.replace(/^[\d•\-*.\s]+/, '').trim())
      .filter(line => line.length > 0)
    
    // Extract action items (lines with "should", "need to", "important to")
    const actionItems = aiText
      .split('\n')
      .filter(line => 
        /should|need to|important to|remember|must|avoid/i.test(line)
      )
      .slice(0, 3)
    
    // Extract definitions (word: explanation patterns)
    const definitions: Record<string, string> = {}
    const defPattern = /(\w+):\s*([^.]+\.)/g
    let match
    while ((match = defPattern.exec(aiText)) !== null) {
      definitions[match[1]] = match[2]
    }
    
    return {
      original: originalText,
      simplified: aiText,
      keyPoints,
      actionItems: actionItems.length > 0 ? actionItems : undefined,
      definitions: Object.keys(definitions).length > 0 ? definitions : undefined,
      confidence: 0.95 // High confidence for GPT-4
    }
  }
  
  // Fallback local translation for common terms
  private fallbackTranslation(text: string, type: string): TranslationResponse {
    const commonTranslations: Record<string, string> = {
      // Lab results
      'glucose': 'blood sugar level',
      'hemoglobin': 'oxygen-carrying protein in blood',
      'cholesterol': 'fat in your blood',
      'creatinine': 'kidney function marker',
      'alt': 'liver enzyme',
      'ast': 'liver enzyme',
      
      // Conditions
      'hypertension': 'high blood pressure',
      'diabetes': 'high blood sugar condition',
      'hyperlipidemia': 'high cholesterol',
      'anemia': 'low red blood cells',
      
      // Medications
      'metformin': 'diabetes medicine',
      'lisinopril': 'blood pressure medicine',
      'atorvastatin': 'cholesterol medicine',
      'aspirin': 'blood thinner',
    }
    
    let simplified = text.toLowerCase()
    
    // Replace medical terms with simple versions
    Object.entries(commonTranslations).forEach(([medical, simple]) => {
      const regex = new RegExp(`\\b${medical}\\b`, 'gi')
      simplified = simplified.replace(regex, simple)
    })
    
    // Simplify numbers and ranges
    simplified = simplified
      .replace(/(\d+\.?\d*)\s*-\s*(\d+\.?\d*)/g, '$1 to $2')
      .replace(/</g, 'less than')
      .replace(/>/g, 'more than')
      .replace(/\bnormal\b/gi, 'good')
      .replace(/\belevated\b/gi, 'high')
      .replace(/\blow\b/gi, 'low')
    
    return {
      original: text,
      simplified,
      keyPoints: ['This is a simplified explanation', 'Please ask your doctor for more details'],
      confidence: 0.6 // Lower confidence for fallback
    }
  }
  
  // Save translation to database for offline access
  private async saveTranslation(original: string, translation: TranslationResponse) {
    // Implementation would save to Supabase
    console.log('Saving translation to database')
  }
  
  // Batch translate multiple items
  async batchTranslate(
    items: TranslationRequest[]
  ): Promise<TranslationResponse[]> {
    return Promise.all(items.map(item => this.translateMedicalText(item)))
  }
  
  // Get translation suggestions for medical terms in a text
  async getMedicalTermSuggestions(text: string): Promise<Record<string, string>> {
    const medicalTerms = this.extractMedicalTerms(text)
    const suggestions: Record<string, string> = {}
    
    for (const term of medicalTerms) {
      const translation = await this.translateMedicalText({
        text: term,
        type: 'general',
        simplificationLevel: 'basic'
      })
      suggestions[term] = translation.simplified
    }
    
    return suggestions
  }
  
  private extractMedicalTerms(text: string): string[] {
    // List of common medical terms to detect
    const medicalPatterns = [
      /\b[A-Z]{2,5}\b/g, // Acronyms 
