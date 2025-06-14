# Split Microservices: A Real-World Case Study

Microservices architecture has become the gold standard for building scalable systems, but knowing when and how to split services is crucial for success. This post explores the strategic decisions behind microservice decomposition through a real-world case study of splitting an Alert service from an Analytics platform.

## The Microservices Question: Why Split?

Before diving into the technical details, let's address the fundamental question: Why do we need microservices in the first place?

### The Benefits of Microservice Architecture

**1. Improved Fault Isolation**
When a large monolithic service fails, it can bring down your entire system. Microservices contain failures, preventing cascading outages that can devastate user experience.

**2. Facilitated Continuous Delivery**
Smaller services enable more frequent deployments with reduced risk. Teams can ship features independently without coordinating massive releases.

**3. Technology Flexibility**
Large monolithic services create technology lock-in. Microservices allow teams to choose the best tools for specific problems and experiment with new frameworks without system-wide impact.

**4. Clear Ownership and Responsibility**
Well-defined service boundaries aligned with business domains enable teams to take full ownership of specific system areas, improving accountability and expertise.

## When Should You Split a Microservice?

The decision to split isn't just about size - it's about identifying the right boundaries. Here are key indicators:

### Domain Boundary Clarity
When you can clearly separate business domains with minimal cross-cutting concerns, it's often a good candidate for splitting. In our case study, Alert functionality had distinct business logic separate from Analytics reporting.

### Team Ownership Patterns
If different teams are working on different parts of the same service and experiencing coordination overhead, splitting might reduce friction.

### Deployment Independence
When parts of your system need to be deployed at different cadences or have different risk profiles, separation becomes valuable.

### Performance Characteristics
Different workloads may have vastly different performance requirements. Alerts need real-time processing, while analytics can tolerate batch processing delays.

## Case Study: Splitting Alert Service from Analytics Platform

Let's examine a real-world example of splitting an Alert service from a larger Analytics platform, demonstrating the complexity and considerations involved.

### The Problem Setup

The Alert and Analytics features were initially co-located in the same repository, creating several challenges:

- **Maintenance overhead**: Different teams working on the same codebase
- **Deployment coupling**: Analytics changes affecting Alert stability
- **Technology constraints**: Framework decisions affecting both domains
- **Unclear ownership**: Shared responsibility leading to gaps in expertise

### System Architecture Overview

The Alert module consisted of three main components:

1. **API Server**: REST endpoints for creating, updating, and managing alerts
2. **Background Processing**: Periodic evaluation and notification system  
3. **Event Consumer**: Message queue consumer for configuration management

## The Migration Strategy: Zero-Downtime Decomposition

The key to successful microservice splitting is maintaining system availability throughout the process. Here's the strategic approach we used:

### Deployment Philosophy

**Tight Coordination Window**: Deployments 1-3 must occur on the same day to minimize code duplication and reduce the maintenance burden of running parallel systems.

**Incremental Migration**: Each deployment phase builds upon the previous one, allowing for validation and rollback at each step.

### Phase 1: API Replication

**Objective**: Create parallel API endpoints without disrupting existing functionality.

**Steps**:
1. **Infrastructure Setup**: Create new environment configurations and secrets management
2. **Code Replication**: Deploy identical Alert APIs to new service domain
3. **Monitoring Setup**: Establish health checks and alerting for the new service
4. **Traffic Switch**: Update frontend to consume new API endpoints

**Key Insight**: Start with the user-facing layer first. This allows you to validate the new service with real traffic while maintaining the ability to quickly rollback.

### Phase 2: Event Processing Migration

**Objective**: Move message queue consumer functionality to the new service.

**Critical Challenge**: Message queue consumer groups maintain offset state. Switching consumers requires careful offset management to prevent message loss or duplication.

**Solution**: Create new consumer group starting from the last unconsumed offset of the old consumer.

Switching with script:
```python
  from confluent_kafka import Consumer as KafkaConsumer, TopicPartition

  def setup_kafka_consumer_with_offset(
      start_offset: int, broker_url: str, topic_name: str, consumer_group: str,
  ):
      kafka_client = KafkaConsumer({
          "bootstrap.servers": broker_url,
          "group.id": consumer_group,
          "enable.auto.commit": False,
      })

      partition_list = kafka_client.list_topics(topic_name).topics[topic_name].partitions.keys()
      print(f"Topic {topic_name} has {len(partition_list)} partitions: {partition_list}")

      assigned_partitions = [TopicPartition(topic_name, pid, start_offset) for pid in partition_list]
      kafka_client.assign(assigned_partitions)
      print(f"Assigned partitions to consumer {consumer_group}")

      for partition in assigned_partitions:
          kafka_client.seek(partition)
      print(f"Seeked all partitions to offset {start_offset}")

      message = kafka_client.poll(1.0)
      if message is None:
          print(f"No message from topic {topic_name} at offset {start_offset}")
      elif message.error():
          print(f"Consumer error: {message.error()}")
      else:
          kafka_client.commit()
          print(f"Committed message from partition {message.partition()} at offset {message.offset()}")
          print(f"Successfully created consumer group {consumer_group}")

      kafka_client.close()
      print(f"Closed consumer {consumer_group}")
```

Switching with tool: Can use Kafka Web UI tools (i.e RedPanda if they support)

### Phase 3: Background Processing Migration

**Objective**: Migrate background processing system for alert evaluation and notifications.

**Complexity**: Background processing systems require careful coordination between job schedulers and worker processes.

**Approach**: Replicate the processing logic while maintaining the existing scheduler infrastructure, then switch processing workspaces atomically.

### Phase 4: Data Migration

**The Most Critical Phase**: Database migration should be the last step. Database migration requires extreme care to maintain data consistency and system availability.

We have two main strategies for database migration:

#### Strategy 1: Backup and Restore (Minimal Downtime)

**Steps**:
1. **Database Preparation**: Create new database with identical schema
2. **Service Preparation**: Configure service to use new database credentials
3. **Traffic Halting**: Temporarily disable write operations (return 503 for non-GET requests and disable event consumer and background jobs)
4. **Data Export/Import**: Use database backup and restore tools for consistent data transfer
5. **Service Switch**: Enable new database and restore full functionality

**Downtime Minimization**: By halting only write operations and maintaining read access, user experience impact is minimized during the brief migration window.

#### Strategy 2: Dual-Write with Replica (Zero Downtime)

**Objective**: Achieve true zero-downtime migration by duplicating writes to both databases during the transition period.

**Prerequisites**: Requires DevOps team collaboration for database replication setup.

**Steps**:
1. **Replica Setup**: DevOps team creates a replica of the source database in the target environment
2. **Dual-Write Implementation**: Modify application code to write to both original and replica databases
3. **Data Consistency Verification**: Implement monitoring to ensure write operations succeed on both databases
4. **Read Traffic Switch**: Gradually shift read operations to the new database while maintaining dual writes
5. **Write Traffic Switch**: Once read operations are stable, switch write operations to target only the new database
6. **Cleanup**: Remove dual-write logic and decommission the original database

**Implementation Example**:
```python
class DualWriteRepository:
    def __init__(self, primary_db, replica_db, migration_mode=True):
        self.primary_db = primary_db
        self.replica_db = replica_db
        self.migration_mode = migration_mode
        
    def create_alert(self, alert_data):
        # Always write to primary first
        result = self.primary_db.create_alert(alert_data)
        
        if self.migration_mode:
            try:
                # Duplicate write to replica
                self.replica_db.create_alert(alert_data)
            except Exception as e:
                # Log error but don't fail the operation
                logger.error(f"Replica write failed: {e}")
                # Could implement retry logic or dead letter queue
                
        return result
        
    def update_alert(self, alert_id, update_data):
        result = self.primary_db.update_alert(alert_id, update_data)
        
        if self.migration_mode:
            try:
                self.replica_db.update_alert(alert_id, update_data)
            except Exception as e:
                logger.error(f"Replica update failed: {e}")
                
        return result
```

**Advantages of Dual-Write Approach**:
- **True Zero Downtime**: No service interruption during migration
- **Gradual Migration**: Can test and validate new database with real traffic
- **Easy Rollback**: Can switch back to original database instantly if issues arise
- **Data Validation**: Opportunity to verify data consistency before full cutover

**Challenges and Considerations**:
- **Code Complexity**: Requires implementing dual-write logic throughout the application
- **Error Handling**: Must handle cases where writes succeed on one database but fail on another
- **Data Consistency**: Need monitoring to ensure both databases stay in sync
- **Performance Impact**: Dual writes can increase latency and resource usage
- **DevOps Coordination**: Requires close collaboration with infrastructure team for replica setup

**Monitoring During Dual-Write Phase**:
```python
class MigrationMonitor:
    def __init__(self, metrics_client):
        self.metrics = metrics_client
        
    def track_dual_write_success(self, operation_type):
        self.metrics.increment(f'migration.dual_write.success.{operation_type}')
        
    def track_dual_write_failure(self, operation_type, database):
        self.metrics.increment(f'migration.dual_write.failure.{operation_type}.{database}')
        
    def track_data_consistency_check(self, consistent: bool):
        status = 'consistent' if consistent else 'inconsistent'
        self.metrics.increment(f'migration.data_consistency.{status}')
```

**Recommendation**: Use Strategy 2 (Dual-Write) for production systems where downtime is not acceptable. Use Strategy 1 (Backup/Restore) for systems that can tolerate brief maintenance windows.

### Phase 5: Cleanup

**Objective**: Remove deprecated code and infrastructure.

**Importance**: Cleanup is often overlooked but crucial for:
- Reducing technical debt
- Eliminating confusion about data sources
- Preventing accidental dependencies on old systems

## Key Takeaways for Microservice Splitting

### 1. Plan for Data Consistency
Data migration is often the most complex part of service splitting. Plan for:
- Schema compatibility during transition periods
- Data consistency verification methods  
- Rollback procedures for data operations

### 2. Maintain Service Contracts
APIs should remain stable during migration. Changes to service contracts can cascade throughout your system.

### 3. Monitor Everything
Comprehensive monitoring during migration helps identify issues before they impact users:
- Application metrics (latency, error rates)
- Infrastructure metrics (CPU, memory, network)
- Business metrics (feature usage, conversion rates)

### 4. Have a Rollback Plan
Every phase should have a clear rollback procedure. Document:
- Decision criteria for rollback
- Step-by-step rollback procedures
- Data recovery methods

### 5. Coordinate Team Communication
Microservice splitting affects multiple teams. Establish:
- Clear communication channels
- Deployment coordination protocols
- Escalation procedures for issues

## Conclusion

Splitting microservices is a complex undertaking that requires careful planning, coordination, and execution. The benefits - improved fault isolation, deployment flexibility, and team autonomy - make it worthwhile when done correctly.

Remember that microservice architecture is not a destination but a journey. Start with clear business boundaries, plan for data consistency, maintain service contracts, and always have a rollback plan.

The key to successful microservice decomposition lies not in the technical implementation details, but in understanding your business domains and designing systems that reflect those boundaries. When done right, microservices enable teams to move faster and build more resilient systems.

**Final thought**: Don't split microservices because it's trendy - split them because it solves real problems in your organization.
The complexity cost is real, but when justified by business needs, the benefits far outweigh the challenges.
