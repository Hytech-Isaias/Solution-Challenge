# Implementation Roadmap & Technical Specifications

## Project Overview

**Duration**: 20 weeks (5 phases)
**Team Size**: 6-8 engineers
**Budget Estimate**: $800K - $1.2M (including infrastructure)

## Team Structure

### Core Team Roles
- **Tech Lead/Architect** (1): Overall architecture and technical decisions
- **Backend Engineers** (3): Microservices development
- **Frontend Engineer** (1): Analytics dashboard and operator interface
- **DevOps Engineer** (1): Infrastructure and deployment
- **Data Engineer** (1): Analytics pipeline and ML models
- **QA Engineer** (1): Testing and quality assurance

## Phase-by-Phase Implementation Plan

### Phase 1: Foundation & Event Infrastructure (Weeks 1-4)

#### Week 1: Project Setup & Infrastructure
**Deliverables:**
```yaml
Infrastructure Setup:
  - AWS account setup and security configuration
  - Kubernetes cluster (EKS) provisioning
  - CI/CD pipeline setup (GitHub Actions)
  - Development environment setup

Event Infrastructure:
  - Kafka cluster deployment (AWS MSK)
  - Schema registry setup
  - Event schema definitions
  - Kafka Connect setup for data pipeline
```

**Technical Tasks:**
```bash
# Infrastructure as Code (Terraform)
terraform/
├── environments/
│   ├── dev/
│   ├── staging/
│   └── prod/
├── modules/
│   ├── eks/
│   ├── kafka/
│   ├── databases/
│   └── monitoring/
└── main.tf

# Kubernetes manifests
k8s/
├── namespaces/
├── services/
├── deployments/
└── ingress/
```

#### Week 2: Database Architecture
**Deliverables:**
```sql
-- PostgreSQL setup and schema
CREATE DATABASE recruitment_platform;

-- Migration scripts using Flyway
V1__create_candidates_table.sql
V2__create_conversation_sessions.sql
V3__create_messages_table.sql
V4__create_recruitment_stages.sql
V5__create_candidate_journey.sql
V6__create_risk_assessments.sql
V7__create_operator_interventions.sql
V8__create_indexes.sql

-- Redis setup
redis/
├── config/
│   ├── redis.conf
│   └── sentinel.conf
└── scripts/
    └── data-structures.redis
```

**Performance Considerations:**
```sql
-- Optimized indexes for frequent queries
CREATE INDEX CONCURRENTLY idx_candidates_phone_number ON candidates(phone_number);
CREATE INDEX CONCURRENTLY idx_candidates_current_state ON candidates(current_state);
CREATE INDEX CONCURRENTLY idx_candidates_last_activity ON candidates(last_activity_at);
CREATE INDEX CONCURRENTLY idx_messages_candidate_created ON messages(candidate_id, created_at);
CREATE INDEX CONCURRENTLY idx_candidate_journey_stage ON candidate_journey(candidate_id, stage_id);

-- Partitioning for large tables
CREATE TABLE messages_2025_q1 PARTITION OF messages
FOR VALUES FROM ('2025-01-01') TO ('2025-04-01');
```

#### Week 3: Core Event System
**Deliverables:**
```typescript
// Event schema definitions
interface BaseEvent {
  eventId: string;
  eventType: string;
  candidateId: string;
  timestamp: Date;
  version: string;
}

interface MessageReceivedEvent extends BaseEvent {
  eventType: 'MESSAGE_RECEIVED';
  messageId: string;
  content: string;
  source: 'whatsapp';
  metadata: {
    phoneNumber: string;
    messageType: 'text' | 'image' | 'document';
  };
}

interface CandidateStateChangedEvent extends BaseEvent {
  eventType: 'CANDIDATE_STATE_CHANGED';
  previousState: string;
  newState: string;
  trigger: string;
  metadata: Record<string, any>;
}

// Event bus implementation
class EventBus {
  private kafka: KafkaJS.Kafka;
  private producer: KafkaJS.Producer;
  private consumers: Map<string, KafkaJS.Consumer>;

  async publishEvent(topic: string, event: BaseEvent): Promise<void> {
    await this.producer.send({
      topic,
      messages: [{
        key: event.candidateId,
        value: JSON.stringify(event),
        timestamp: event.timestamp.getTime().toString()
      }]
    });
  }

  async subscribeToTopic(topic: string, handler: EventHandler): Promise<void> {
    const consumer = this.kafka.consumer({ groupId: `${topic}-group` });
    await consumer.subscribe({ topic });
    await consumer.run({
      eachMessage: async ({ message }) => {
        const event = JSON.parse(message.value.toString());
        await handler.handle(event);
      }
    });
  }
}
```

#### Week 4: Enhanced Webhook Service
**Deliverables:**
```typescript
// Webhook service with event publishing
@Controller('/webhook')
export class WebhookController {
  constructor(
    private eventBus: EventBus,
    private validationService: ValidationService,
    private rateLimitService: RateLimitService
  ) {}

  @Post('/whatsapp')
  async handleWhatsAppWebhook(@Body() webhook: WhatsAppWebhook): Promise<void> {
    // Validate webhook signature
    await this.validationService.validateSignature(webhook);
    
    // Rate limiting
    await this.rateLimitService.checkLimit(webhook.sender);
    
    // Process messages
    for (const message of webhook.messages) {
      const event: MessageReceivedEvent = {
        eventId: generateUUID(),
        eventType: 'MESSAGE_RECEIVED',
        candidateId: await this.getCandidateId(message.from),
        timestamp: new Date(),
        version: '1.0',
        messageId: message.id,
        content: message.text.body,
        source: 'whatsapp',
        metadata: {
          phoneNumber: message.from,
          messageType: message.type
        }
      };
      
      await this.eventBus.publishEvent('candidate-interactions', event);
    }
  }

  private async getCandidateId(phoneNumber: string): Promise<string> {
    let candidate = await this.candidateService.findByPhone(phoneNumber);
    if (!candidate) {
      candidate = await this.candidateService.create({ phoneNumber });
      
      // Publish candidate created event
      const event: CandidateCreatedEvent = {
        eventId: generateUUID(),
        eventType: 'CANDIDATE_CREATED',
        candidateId: candidate.id,
        timestamp: new Date(),
        version: '1.0',
        phoneNumber
      };
      
      await this.eventBus.publishEvent('candidate-lifecycle', event);
    }
    return candidate.id;
  }
}
```

**Testing Strategy:**
```typescript
// End-to-end webhook testing
describe('Webhook Integration Tests', () => {
  it('should process WhatsApp message and publish events', async () => {
    const webhook = createMockWhatsAppWebhook();
    const response = await request(app)
      .post('/webhook/whatsapp')
      .send(webhook)
      .expect(200);
    
    // Verify event was published
    const publishedEvents = await testEventCapture.getEvents();
    expect(publishedEvents).toHaveLength(1);
    expect(publishedEvents[0].eventType).toBe('MESSAGE_RECEIVED');
  });
});
```

### Phase 2: Real-time Analytics & Observability (Weeks 5-8)

#### Week 5: Analytics Service Development
**Deliverables:**
```go
// High-performance analytics service in Go
package analytics

import (
    "context"
    "time"
    "github.com/influxdata/influxdb-client-go/v2"
)

type AnalyticsService struct {
    influxClient influxdb2.Client
    writeAPI     api.WriteAPIBlocking
}

type FunnelMetrics struct {
    TotalCandidates    int                 `json:"total_candidates"`
    StageCounts        map[string]int      `json:"stage_counts"`
    ConversionRates    map[string]float64  `json:"conversion_rates"`
    DropOffPoints      []DropOffPoint      `json:"drop_off_points"`
    LastUpdated        time.Time           `json:"last_updated"`
}

func (s *AnalyticsService) ProcessCandidateEvent(event CandidateEvent) error {
    // Write to InfluxDB for time-series analytics
    point := influxdb2.NewPoint(
        "candidate_interactions",
        map[string]string{
            "candidate_id": event.CandidateID,
            "event_type":   event.EventType,
            "stage":        event.Stage,
        },
        map[string]interface{}{
            "duration": event.Duration,
            "success":  event.Success,
        },
        event.Timestamp,
    )
    
    return s.writeAPI.WritePoint(context.Background(), point)
}

func (s *AnalyticsService) GetRealTimeFunnelMetrics() (*FunnelMetrics, error) {
    query := `
        from(bucket: "recruitment")
        |> range(start: -24h)
        |> filter(fn: (r) => r._measurement == "candidate_interactions")
        |> group(columns: ["stage"])
        |> count()
    `
    
    result, err := s.influxClient.QueryAPI("recruitment").Query(context.Background(), query)
    if err != nil {
        return nil, err
    }
    
    // Process results and calculate metrics
    return s.buildFunnelMetrics(result), nil
}
```

#### Week 6: Real-time Dashboard Development
**Deliverables:**
```typescript
// React dashboard with real-time updates
import React, { useState, useEffect } from 'react';
import { LineChart, BarChart, PieChart } from 'recharts';
import { useWebSocket } from './hooks/useWebSocket';

interface DashboardProps {}

export const RecruitmentDashboard: React.FC<DashboardProps> = () => {
  const [funnelData, setFunnelData] = useState<FunnelMetrics | null>(null);
  const [realTimeEvents, setRealTimeEvents] = useState<Event[]>([]);
  
  // WebSocket connection for real-time updates
  const { messages } = useWebSocket('ws://localhost:8080/ws/dashboard');
  
  useEffect(() => {
    // Process real-time messages
    if (messages.length > 0) {
      const latestMessage = messages[messages.length - 1];
      if (latestMessage.type === 'FUNNEL_UPDATE') {
        setFunnelData(latestMessage.data);
      }
    }
  }, [messages]);
  
  return (
    <div className="dashboard-container">
      <div className="metrics-grid">
        <MetricCard 
          title="Active Candidates" 
          value={funnelData?.totalCandidates || 0}
          trend="+12% from yesterday"
        />
        
        <FunnelVisualization data={funnelData?.stageCounts} />
        
        <ConversionRatesChart data={funnelData?.conversionRates} />
        
        <DropOffAnalysis data={funnelData?.dropOffPoints} />
      </div>
      
      <div className="real-time-feed">
        <h3>Live Activity Feed</h3>
        <EventStream events={realTimeEvents} />
      </div>
    </div>
  );
};

// WebSocket hook for real-time updates
export const useWebSocket = (url: string) => {
  const [socket, setSocket] = useState<WebSocket | null>(null);
  const [messages, setMessages] = useState<any[]>([]);
  
  useEffect(() => {
    const ws = new WebSocket(url);
    
    ws.onmessage = (event) => {
      const message = JSON.parse(event.data);
      setMessages(prev => [...prev, message]);
    };
    
    setSocket(ws);
    
    return () => ws.close();
  }, [url]);
  
  return { messages, socket };
};
```

#### Week 7: State Management Service
**Deliverables:**
```typescript
// Comprehensive state management with Redis
export class StateManagementService {
  private redis: Redis;
  private eventBus: EventBus;
  
  constructor(redis: Redis, eventBus: EventBus) {
    this.redis = redis;
    this.eventBus = eventBus;
  }
  
  async updateCandidateState(
    candidateId: string, 
    newState: string, 
    context?: Record<string, any>
  ): Promise<void> {
    const currentState = await this.getCurrentState(candidateId);
    
    // Validate state transition
    if (!this.isValidTransition(currentState, newState)) {
      throw new Error(`Invalid state transition: ${currentState} -> ${newState}`);
    }
    
    // Update state in Redis
    const sessionKey = `candidate_session:${candidateId}`;
    await this.redis.hmset(sessionKey, {
      currentState: newState,
      lastUpdated: new Date().toISOString(),
      context: JSON.stringify(context || {})
    });
    
    // Publish state change event
    const event: CandidateStateChangedEvent = {
      eventId: generateUUID(),
      eventType: 'CANDIDATE_STATE_CHANGED',
      candidateId,
      timestamp: new Date(),
      version: '1.0',
      previousState: currentState,
      newState,
      trigger: 'system',
      metadata: context || {}
    };
    
    await this.eventBus.publishEvent('candidate-state-changes', event);
  }
  
  async getCandidateContext(candidateId: string): Promise<CandidateContext> {
    const sessionKey = `candidate_session:${candidateId}`;
    const sessionData = await this.redis.hgetall(sessionKey);
    
    return {
      candidateId,
      currentState: sessionData.currentState,
      lastActivity: new Date(sessionData.lastActivity),
      isOperatorActive: sessionData.isOperatorActive === 'true',
      operatorId: sessionData.operatorId || null,
      contextData: JSON.parse(sessionData.context || '{}')
    };
  }
  
  private isValidTransition(from: string, to: string): boolean {
    const validTransitions = {
      'initial_contact': ['screening_questions', 'disqualified'],
      'screening_questions': ['technical_challenge', 'cultural_fit', 'disqualified'],
      'technical_challenge': ['technical_review', 'cultural_fit', 'disqualified'],
      'technical_review': ['cultural_fit', 'final_review', 'disqualified'],
      'cultural_fit': ['final_review', 'disqualified'],
      'final_review': ['hired', 'disqualified'],
      'hired': [],
      'disqualified': []
    };
    
    return validTransitions[from]?.includes(to) || false;
  }
}

// State machine configuration
export const recruitmentStateMachine = {
  initial: 'initial_contact',
  states: {
    initial_contact: {
      on: {
        BASIC_INFO_COMPLETE: 'screening_questions',
        DISQUALIFY: 'disqualified'
      }
    },
    screening_questions: {
      on: {
        SCREENING_PASSED: 'technical_challenge',
        SKIP_TO_CULTURAL: 'cultural_fit',
        DISQUALIFY: 'disqualified'
      }
    },
    technical_challenge: {
      on: {
        CHALLENGE_SUBMITTED: 'technical_review',
        SKIP_TO_CULTURAL: 'cultural_fit',
        DISQUALIFY: 'disqualified'
      }
    },
    technical_review: {
      on: {
        REVIEW_PASSED: 'cultural_fit',
        FAST_TRACK: 'final_review',
        DISQUALIFY: 'disqualified'
      }
    },
    cultural_fit: {
      on: {
        CULTURAL_PASSED: 'final_review',
        DISQUALIFY: 'disqualified'
      }
    },
    final_review: {
      on: {
        HIRE: 'hired',
        DISQUALIFY: 'disqualified'
      }
    },
    hired: { type: 'final' },
    disqualified: { type: 'final' }
  }
};
```

#### Week 8: Monitoring & Alerting Setup
**Deliverables:**
```yaml
# Prometheus configuration
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "recruitment_platform_rules.yml"

scrape_configs:
  - job_name: 'webhook-service'
    static_configs:
      - targets: ['webhook-service:3000']
    metrics_path: /metrics
    scrape_interval: 10s

  - job_name: 'analytics-service'
    static_configs:
      - targets: ['analytics-service:8080']
    metrics_path: /metrics
    scrape_interval: 30s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

# Alert rules
groups:
  - name: recruitment_platform
    rules:
      - alert: HighMessageLatency
        expr: histogram_quantile(0.95, rate(message_processing_duration_seconds_bucket[5m])) > 0.5
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High message processing latency"
          description: "95th percentile latency is {{ $value }}s"

      - alert: CandidateDropOffSpike
        expr: rate(candidate_drop_off_total[5m]) > 0.1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Unusual candidate drop-off rate"
          description: "Drop-off rate is {{ $value }} per second"

      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service is down"
          description: "{{ $labels.job }} service is not responding"
```

```typescript
// Application metrics collection
import prometheus from 'prom-client';

export class MetricsCollector {
  private messageProcessingDuration: prometheus.Histogram;
  private candidateStateTransitions: prometheus.Counter;
  private activeConversations: prometheus.Gauge;
  
  constructor() {
    this.messageProcessingDuration = new prometheus.Histogram({
      name: 'message_processing_duration_seconds',
      help: 'Duration of message processing',
      labelNames: ['message_type', 'service']
    });
    
    this.candidateStateTransitions = new prometheus.Counter({
      name: 'candidate_state_transitions_total',
      help: 'Total number of candidate state transitions',
      labelNames: ['from_state', 'to_state']
    });
    
    this.activeConversations = new prometheus.Gauge({
      name: 'active_conversations_total',
      help: 'Current number of active conversations'
    });
  }
  
  recordMessageProcessing(duration: number, messageType: string, service: string): void {
    this.messageProcessingDuration
      .labels(messageType, service)
      .observe(duration);
  }
  
  recordStateTransition(fromState: string, toState: string): void {
    this.candidateStateTransitions
      .labels(fromState, toState)
      .inc();
  }
  
  updateActiveConversations(count: number): void {
    this.activeConversations.set(count);
  }
}
```

### Phase 3: Risk Scoring & Proactive Intervention (Weeks 9-12)

#### Week 9: Risk Scoring ML Model
**Deliverables:**
```python
# Risk scoring service with ML models
import pandas as pd
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split
from sklearn.metrics import classification_report
import joblib
from datetime import datetime, timedelta

class CandidateRiskScorer:
    def __init__(self):
        self.model = None
        self.feature_columns = [
            'time_since_last_message',
            'total_messages_sent',
            'response_time_avg',
            'current_stage_duration',
            'previous_stage_completion_rate',
            'message_length_avg',
            'weekend_activity',
            'time_of_day_activity'
        ]
    
    def prepare_features(self, candidate_data: dict) -> np.ndarray:
        """Extract features for risk prediction"""
        features = []
        
        # Time-based features
        last_activity = datetime.fromisoformat(candidate_data['last_activity'])
        time_since_last = (datetime.now() - last_activity).total_seconds() / 3600  # hours
        features.append(time_since_last)
        
        # Engagement features
        features.append(candidate_data.get('total_messages', 0))
        features.append(candidate_data.get('avg_response_time', 0))
        
        # Stage progression features
        stage_duration = candidate_data.get('current_stage_duration', 0)
        features.append(stage_duration)
        features.append(candidate_data.get('stage_completion_rate', 0))
        
        # Communication patterns
        features.append(candidate_data.get('avg_message_length', 0))
        features.append(candidate_data.get('weekend_activity_score', 0))
        features.append(candidate_data.get('time_of_day_score', 0))
        
        return np.array(features).reshape(1, -1)
    
    def predict_risk(self, candidate_data: dict) -> dict:
        """Predict risk score and factors for a candidate"""
        if self.model is None:
            raise ValueError("Model not trained yet")
        
        features = self.prepare_features(candidate_data)
        risk_probability = self.model.predict_proba(features)[0][1]  # Probability of drop-off
        
        # Calculate feature importance for this prediction
        feature_importance = self.model.feature_importances_
        top_risk_factors = []
        
        for i, importance in enumerate(feature_importance):
            if importance > 0.1:  # Only significant factors
                top_risk_factors.append({
                    'factor': self.feature_columns[i],
                    'importance': importance,
                    'value': features[0][i]
                })
        
        top_risk_factors.sort(key=lambda x: x['importance'], reverse=True)
        
        return {
            'candidate_id': candidate_data['candidate_id'],
            'risk_score': float(risk_probability),
            'risk_level': self._categorize_risk(risk_probability),
            'top_risk_factors': top_risk_factors[:3],
            'recommended_actions': self._recommend_actions(risk_probability, top_risk_factors),
            'assessed_at': datetime.now().isoformat()
        }
    
    def _categorize_risk(self, score: float) -> str:
        if score >= 0.8:
            return 'HIGH'
        elif score >= 0.6:
            return 'MEDIUM'
        elif score >= 0.3:
            return 'LOW'
        else:
            return 'MINIMAL'
    
    def _recommend_actions(self, risk_score: float, risk_factors: list) -> list:
        actions = []
        
        if risk_score >= 0.8:
            actions.append('IMMEDIATE_INTERVENTION')
            actions.append('PERSONAL_OUTREACH')
        elif risk_score >= 0.6:
            actions.append('PROACTIVE_MESSAGE')
            actions.append('SCHEDULE_CHECK_IN')
        elif risk_score >= 0.3:
            actions.append('MONITOR_CLOSELY')
        
        # Factor-specific recommendations
        for factor in risk_factors:
            if factor['factor'] == 'time_since_last_message' and factor['value'] > 24:
                actions.append('SEND_REMINDER')
            elif factor['factor'] == 'response_time_avg' and factor['value'] > 3600:
                actions.append('SIMPLIFY_QUESTIONS')
        
        return list(set(actions))  # Remove duplicates

# FastAPI service for risk scoring
from fastapi import FastAPI, BackgroundTasks
from pydantic import BaseModel

app = FastAPI()
risk_scorer = CandidateRiskScorer()

class RiskAssessmentRequest(BaseModel):
    candidate_id: str
    candidate_data: dict

@app.post("/assess-risk")
async def assess_candidate_risk(
    request: RiskAssessmentRequest,
    background_tasks: BackgroundTasks
):
    risk_assessment = risk_scorer.predict_risk(request.candidate_data)
    
    # Trigger notifications if high risk
    if risk_assessment['risk_score'] >= 0.6:
        background_tasks.add_task(
            trigger_intervention_alert,
            risk_assessment
        )
    
    return risk_assessment

async def trigger_intervention_alert(risk_assessment: dict):
    """Background task to send notifications"""
    # Publish risk event to Kafka
    event = {
        'eventType': 'CANDIDATE_AT_RISK',
        'candidateId': risk_assessment['candidate_id'],
        'riskScore': risk_assessment['risk_score'],
        'riskLevel': risk_assessment['risk_level'],
        'recommendedActions': risk_assessment['recommended_actions'],
        'timestamp': datetime.now().isoformat()
    }
    
    # Send to event bus (implementation depends on your event bus)
    await event_bus.publish('candidate-risk-alerts', event)
```

#### Week 10: Automated Monitoring System
**Deliverables:**
```typescript
// Background monitoring service
export class CandidateMonitoringService {
  private riskScoringClient: RiskScoringClient;
  private candidateRepository: CandidateRepository;
  private eventBus: EventBus;
  private schedulerService: SchedulerService;
  
  constructor(
    riskScoringClient: RiskScoringClient,
    candidateRepository: CandidateRepository,
    eventBus: EventBus,
    schedulerService: SchedulerService
  ) {
    this.riskScoringClient = riskScoringClient;
    this.candidateRepository = candidateRepository;
    this.eventBus = eventBus;
    this.schedulerService = schedulerService;
  }
  
  async startMonitoring(): Promise<void> {
    // Schedule periodic risk assessments
    this.schedulerService.scheduleRecurring(
      'candidate-risk-assessment',
      '*/15 * * * *', // Every 15 minutes
      () => this.performRiskAssessment()
    );
    
    // Schedule inactivity checks
    this.schedulerService.scheduleRecurring(
      'inactivity-check',
      '0 */6 * * *', // Every 6 hours
      () => this.checkInactiveCandidates()
    );
  }
  
  private async performRiskAssessment(): Promise<void> {
    const activeCandidates = await this.candidateRepository.findActiveCandidates();
    
    for (const candidate of activeCandidates) {
      try {
        const candidateData = await this.prepareCandidateData(candidate);
        const riskAssessment = await this.riskScoringClient.assessRisk(candidateData);
        
        // Store risk assessment
        await this.candidateRepository.updateRiskScore(
          candidate.id,
          riskAssessment.riskScore
        );
        
        // Trigger interventions if needed
        if (riskAssessment.riskLevel === 'HIGH') {
          await this.triggerHighRiskIntervention(candidate, riskAssessment);
        } else if (riskAssessment.riskLevel === 'MEDIUM') {
          await this.triggerMediumRiskIntervention(candidate, riskAssessment);
        }
        
      } catch (error) {
        console.error(`Risk assessment failed for candidate ${candidate.id}:`, error);
      }
    }
  }
  
  private async prepareCandidateData(candidate: Candidate): Promise<CandidateData> {
    const messages = await this.candidateRepository.getCandidateMessages(candidate.id);
    const stageHistory = await this.candidateRepository.getStageHistory(candidate.id);
    
    return {
      candidate_id: candidate.id,
      last_activity: candidate.lastActivityAt.toISOString(),
      total_messages: messages.length,
      avg_response_time: this.calculateAverageResponseTime(messages),
      current_stage_duration: this.calculateCurrentStageDuration(stageHistory),
      stage_completion_rate: this.calculateStageCompletionRate(stageHistory),
      avg_message_length: this.calculateAverageMessageLength(messages),
      weekend_activity_score: this.calculateWeekendActivityScore(messages),
      time_of_day_score: this.calculateTimeOfDayScore(messages)
    };
  }
  
  private async triggerHighRiskIntervention(
    candidate: Candidate,
    riskAssessment: RiskAssessment
  ): Promise<void> {
    const event: CandidateAtRiskEvent = {
      eventId: generateUUID(),
      eventType: 'CANDIDATE_AT_RISK',
      candidateId: candidate.id,
      timestamp: new Date(),
      version: '1.0',
      riskScore: riskAssessment.riskScore,
      riskLevel: 'HIGH',
      riskFactors: riskAssessment.topRiskFactors.map(f => f.factor),
      recommendedActions: riskAssessment.recommendedActions,
      urgency: 'IMMEDIATE'
    };
    
    await this.eventBus.publishEvent('candidate-risk-alerts', event);
  }
  
  private async checkInactiveCandidates(): Promise<void> {
    const inactiveSince = new Date(Date.now() - 24 * 60 * 60 * 1000); // 24 hours ago
    const inactiveCandidates = await this.candidateRepository.findInactiveSince(inactiveSince);
    
    for (const candidate of inactiveCandidates) {
      const event: CandidateInactiveEvent = {
        eventId: generateUUID(),
        eventType: 'CANDIDATE_INACTIVE',
        candidateId: candidate.id,
        timestamp: new Date(),
        version: '1.0',
        inactiveDuration: Date.now() - candidate.lastActivityAt.getTime(),
        lastStage: candidate.currentState
      };
      
      await this.eventBus.publishEvent('candidate-inactivity', event);
    }
  }
}
```

#### Week 11: Notification Service Development
**Deliverables:**
```typescript
// Multi-channel notification service
export class NotificationService {
  private slackClient: SlackWebClient;
  private emailService: EmailService;
  private eventBus: EventBus;
  private notificationRules: NotificationRules;
  
  constructor(
    slackClient: SlackWebClient,
    emailService: EmailService,
    eventBus: EventBus,
    notificationRules: NotificationRules
  ) {
    this.slackClient = slackClient;
    this.emailService = emailService;
    this.eventBus = eventBus;
    this.notificationRules = notificationRules;
    
    this.setupEventListeners();
  }
  
  private setupEventListeners(): void {
    this.eventBus.subscribe('candidate-risk-alerts', async (event) => {
      await this.handleRiskAlert(event as CandidateAtRiskEvent);
    });
    
    this.eventBus.subscribe('candidate-inactivity', async (event) => {
      await this.handleInactivityAlert(event as CandidateInactiveEvent);
    });
    
    this.eventBus.subscribe('system-alerts', async (event) => {
      await this.handleSystemAlert(event as SystemAlertEvent);
    });
  }
  
  private async handleRiskAlert(event: CandidateAtRiskEvent): Promise<void> {
    const candidate = await this.getCandidateDetails(event.candidateId);
    const rules = this.notificationRules.getRiskAlertRules(event.riskLevel);
    
    for (const rule of rules) {
      switch (rule.channel) {
        case 'slack':
          await this.sendSlackRiskAlert(candidate, event, rule);
          break;
        case 'email':
          await this.sendEmailRiskAlert(candidate, event, rule);
          break;
        case 'dashboard':
          await this.sendDashboardAlert(candidate, event, rule);
          break;
      }
    }
  }
  
  private async sendSlackRiskAlert(
    candidate: Candidate,
    event: CandidateAtRiskEvent,
    rule: NotificationRule
  ): Promise<void> {
    const blocks = [
      {
        type: 'header',
        text: {
          type: 'plain_text',
          text: `🚨 ${event.riskLevel} Risk Candidate Alert`
        }
      },
      {
        type: 'section',
        fields: [
          {
            type: 'mrkdwn',
            text: `*Candidate:* ${candidate.firstName} ${candidate.lastName}`
          },
          {
            type: 'mrkdwn',
            text: `*Phone:* ${candidate.phoneNumber}`
          },
          {
            type: 'mrkdwn',
            text: `*Current Stage:* ${candidate.currentState}`
          },
          {
            type: 'mrkdwn',
            text: `*Risk Score:* ${(event.riskScore * 100).toFixed(1)}%`
          }
        ]
      },
      {
        type: 'section',
        text: {
          type: 'mrkdwn',
          text: `*Risk Factors:*\n${event.riskFactors.map(f => `• ${f}`).join('\n')}`
        }
      },
      {
        type: 'section',
        text: {
          type: 'mrkdwn',
          text: `*Recommended Actions:*\n${event.recommendedActions.map(a => `• ${a}`).join('\n')}`
        }
      },
      {
        type: 'actions',
        elements: [
          {
            type: 'button',
            text: {
              type: 'plain_text',
              text: 'Take Over Conversation'
            },
            style: 'primary',
            value: `takeover_${candidate.id}`,
            action_id: 'takeover_conversation'
          },
          {
            type: 'button',
            text: {
              type: 'plain_text',
              text: 'Send Proactive Message'
            },
            value: `proactive_${candidate.id}`,
            action_id: 'send_proactive_message'
          },
          {
            type: 'button',
            text: {
              type: 'plain_text',
              text: 'View Dashboard'
            },
            url: `${process.env.DASHBOARD_URL}/candidates/${candidate.id}`,
            action_id: 'view_dashboard'
          }
        ]
      }
    ];
    
    await this.slackClient.chat.postMessage({
      channel: rule.target,
      blocks,
      text: `${event.riskLevel} risk candidate alert for ${candidate.firstName} ${candidate.lastName}`
    });
  }
  
  private async sendEmailRiskAlert(
    candidate: Candidate,
    event: CandidateAtRiskEvent,
    rule: NotificationRule
  ): Promise<void> {
    const emailTemplate = await this.loadEmailTemplate('risk-alert');
    const emailContent = emailTemplate.render({
      candidate,
      riskScore: event.riskScore,
      riskLevel: event.riskLevel,
      riskFactors: event.riskFactors,
      recommendedActions: event.recommendedActions,
      dashboardUrl: `${process.env.DASHBOARD_URL}/candidates/${candidate.id}`
    });
    
    await this.emailService.send({
      to: rule.target,
      subject: `${event.riskLevel} Risk Alert: ${candidate.firstName} ${candidate.lastName}`,
      html: emailContent,
      priority: event.urgency === 'IMMEDIATE' ? 'high' : 'normal'
    });
  }
}

// Notification rules configuration
export class NotificationRules {
  private rules: Map<string, NotificationRule[]>;
  
  constructor() {
    this.rules = new Map([
      ['HIGH_RISK', [
        {
          channel: 'slack',
          target: '#recruitment-alerts',
          urgency: 'immediate',
          escalation: {
            delay: 300, // 5 minutes
            nextRule: 'HIGH_RISK_ESCALATION'
          }
        },
        {
          channel: 'email',
          target: 'recruitment-lead@company.com',
          urgency: 'immediate'
        }
      ]],
      ['MEDIUM_RISK', [
        {
          channel: 'slack',
          target: '#recruitment-alerts',
          urgency: 'normal'
        },
        {
          channel: 'dashboard',
          target: 'risk-alerts-panel',
          urgency: 'normal'
        }
      ]],
      ['LOW_RISK', [
        {
          channel: 'dashboard',
          target: 'risk-alerts-panel',
          urgency: 'low'
        }
      ]]
    ]);
  }
  
  getRiskAlertRules(riskLevel: string): NotificationRule[] {
    return this.rules.get(riskLevel) || [];
  }
}
```

#### Week 12: Integration Testing & Optimization
**Deliverables:**
```typescript
// Comprehensive integration tests
describe('Risk Detection & Notification Integration', () => {
  let testEnvironment: TestEnvironment;
  let mockSlackClient: jest.Mocked<SlackWebClient>;
  let mockEventBus: jest.Mocked<EventBus>;
  
  beforeEach(async () => {
    testEnvironment = await setupTestEnvironment();
    mockSlackClient = createMockSlackClient();
    mockEventBus = createMockEventBus();
  });
  
  describe('End-to-End Risk Detection Flow', () => {
    it('should detect high-risk candidate and trigger notifications', async () => {
      // Setup test candidate with high-risk characteristics
      const candidate = await testEnvironment.createCandidate({
        phoneNumber: '+1234567890',
        firstName: 'John',
        lastName: 'Doe',
        currentState: 'technical_challenge',
        lastActivityAt: new Date(Date.now() - 25 * 60 * 60 * 1000) // 25 hours ago
      });
      
      // Add message history indicating declining engagement
      await testEnvironment.addCandidateMessages(candidate.id, [
        { content: 'Looking forward to the challenge!', timestamp: hoursAgo(24) },
        { content: 'This seems difficult...', timestamp: hoursAgo(20) },
        { content: 'Not sure about this', timestamp: hoursAgo(18) }
      ]);
      
      // Trigger risk assessment
      const monitoringService = testEnvironment.getMonitoringService();
      await monitoringService.performRiskAssessment();
      
      // Verify risk detection
      const publishedEvents = mockEventBus.getPublishedEvents();
      const riskEvent = publishedEvents.find(e => e.eventType === 'CANDIDATE_AT_RISK');
      
      expect(riskEvent).toBeDefined();
      expect(riskEvent.candidateId).toBe(candidate.id);
      expect(riskEvent.riskLevel).toBe('HIGH');
      expect(riskEvent.riskScore).toBeGreaterThan(0.8);
      
      // Verify Slack notification was sent
      expect(mockSlackClient.chat.postMessage).toHaveBeenCalledWith(
        expect.objectContaining({
          channel: '#recruitment-alerts',
          blocks: expect.arrayContaining([
            expect.objectContaining({
              type: 'header',
              text: expect.objectContaining({
                text: expect.stringContaining('HIGH Risk Candidate Alert')
              })
            })
          ])
        })
      );
    });
    
    it('should handle medium-risk candidates appropriately', async () => {
      const candidate = await testEnvironment.createCandidate({
        phoneNumber: '+1234567891',
        currentState: 'screening_questions',
        lastActivityAt: new Date(Date.now() - 12 * 60 * 60 * 1000) // 12 hours ago
      });
      
      // Add moderate engagement pattern
      await testEnvironment.addCandidateMessages(candidate.id, [
        { content: 'Thanks for the opportunity', timestamp: hoursAgo(12) },
        { content: 'I\'ll get back to you soon', timestamp: hoursAgo(8) }
      ]);
      
      const monitoringService = testEnvironment.getMonitoringService();
      await monitoringService.performRiskAssessment();
      
      const publishedEvents = mockEventBus.getPublishedEvents();
      const riskEvent = publishedEvents.find(e => e.eventType === 'CANDIDATE_AT_RISK');
      
      if (riskEvent && riskEvent.riskLevel === 'MEDIUM') {
        // Should trigger dashboard notification but not immediate Slack alert
        expect(mockSlackClient.chat.postMessage).toHaveBeenCalledWith(
          expect.objectContaining({
            channel: '#recruitment-alerts'
          })
        );
      }
    });
  });
  
  describe('Notification Escalation', () => {
    it('should escalate unacknowledged high-risk alerts', async () => {
      // Create high-risk scenario
      const candidate = await testEnvironment.createCandidate({
        phoneNumber: '+1234567892',
        currentState: 'final_review',
        lastActivityAt: new Date(Date.now() - 30 * 60 * 60 * 1000) // 30 hours ago
      });
      
      // Trigger initial alert
      await testEnvironment.triggerRiskAlert(candidate.id, 'HIGH');
      
      // Wait for escalation delay (mocked)
      await testEnvironment.advanceTime(5 * 60 * 1000); // 5 minutes
      
      // Verify escalation was triggered
      const escalationEvents = mockEventBus.getPublishedEvents()
        .filter(e => e.eventType === 'ALERT_ESCALATION');
      
      expect(escalationEvents).toHaveLength(1);
      expect(escalationEvents[0].candidateId).toBe(candidate.id);
    });
  });
  
  describe('Performance Under Load', () => {
    it('should handle 1000 concurrent risk assessments', async () => {
      const candidates = await Promise.all(
        Array.from({ length: 1000 }, (_, i) => 
          testEnvironment.createCandidate({
            phoneNumber: `+123456${i.toString().padStart(4, '0')}`,
            currentState: 'technical_challenge'
          })
        )
      );
      
      const startTime = Date.now();
      const monitoringService = testEnvironment.getMonitoringService();
      
      // Process all candidates
      await monitoringService.performRiskAssessment();
      
      const processingTime = Date.now() - startTime;
      
      // Should complete within reasonable time (e.g., 30 seconds)
      expect(processingTime).toBeLessThan(30000);
      
      // Verify all candidates were processed
      const updatedCandidates = await testEnvironment.getCandidates();
      const candidatesWithRiskScores = updatedCandidates.filter(c => c.riskScore !== null);
      
      expect(candidatesWithRiskScores).toHaveLength(1000);
    });
  });
});
```

This comprehensive implementation plan provides:

1. **Clear week-by-week deliverables** with specific technical components
2. **Production-ready code examples** showing real implementation details
3. **Comprehensive testing strategy** including integration and performance tests
4. **Scalable architecture** designed to handle high volumes
5. **Monitoring and observability** built-in from the start
6. **Risk management** through gradual rollout and testing

The solution transforms the "black box" into a fully observable, controllable system while maintaining high performance and reliability standards.
