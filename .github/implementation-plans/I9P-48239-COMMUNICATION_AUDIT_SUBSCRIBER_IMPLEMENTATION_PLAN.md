Implementation Plan: Communication Audit Event Subscriber
JIRA: I9P-48239 Repos affected: 8149_US_i9-audit-handler_WSA (primary), 1574_US_ES-ONB-AUDIT-API_WSA (downstream — new audit_communications table)


________________


1. Summary
Add a new PubSub subscriber to i9-audit-handler that consumes communication (email/SMS) audit events, decrypts PII fields using the existing PubSubDecryptionService, and posts the result to es-onb-audit-api's /internal/eev/es-onb-audit-api/v1/audit-data endpoint. The communications details (channel/subject/body) are persisted to a new, separate audit_communications table in es-onb-audit-api rather than nested inside the generic additional_info JSON column.


Additionally, rename the existing AuditEventSubscriber → EverifyAuditEventSubscriber to disambiguate it from the new communication subscriber.


________________


2. Confirmed decisions
Question
	Decision
	Where does the DEK gcsBucketPath come from?
	It is included as a field on the raw inbound message payload itself (not a separate lookup/attribute). The subscriber reads it directly off the deserialized message before decrypting the PII fields.
	Are packetId / employerId / employeeId / locationId UUIDs?
	Yes — same typing as the existing AuditEvent DTO (UUID).
	Where do communications + decrypted PII fields live downstream?
	Separate audit_communications table in es-onb-audit-api (not nested inside additional_info) — requires a cross-repo change.
	Is the new subscription in the same GCP project as the Everify one?
	Yes — ews-es-i9-npe-00c4 (dev/qa) and ews-es-i9-prd-e87d (uat/prod), identical to the existing spring.cloud.gcp.project-id values. No new GCP project wiring needed, only a new subscription name per environment.
	

Still to confirm with the producer team before coding: the exact JSON key name used for the DEK path field on the inbound message (assumed gcsBucketPath below — rename trivially if different).


________________


3. Rename AuditEventSubscriber → EverifyAuditEventSubscriber
File
	Change
	subscriber/AuditEventSubscriber.java → subscriber/EverifyAuditEventSubscriber.java
	Rename class + file. Constructor @Value / @ConditionalOnProperty keys (audit.everify-event-subscriber.*) stay unchanged — they already use "everify" naming.
	test/.../AuditEventSubscriberTest.java → EverifyAuditEventSubscriberTest.java
	Rename class + all references.
	

No other files reference AuditEventSubscriber by name (verified via workspace search).


________________


4. New enum values (i9-audit-handler)
* AuditDomain: add COMMUNICATION("communication").
* AuditEventType: add COMMUNICATION("communication").
* AuditUserType: add HR("HR").


(Case-insensitive enum matching + unknown-enum-as-default are already enabled globally via JacksonPubSubMessageConverterConfiguration.)


________________


5. New DTOs (i9-audit-handler)
5.1 dto/Communication.java (new)
@Data @Builder @NoArgsConstructor @AllArgsConstructor
@JsonIgnoreProperties(ignoreUnknown = true)
public class Communication {
    private String channel;   // EMAIL, SMS
    private String subject;   // not encrypted
    private String body;      // encrypted at the source — decrypt before building the API request
}
5.2 Inbound event DTO
Reuse AuditEvent, adding:


private List<Communication> communications;
private String gcsBucketPath;   // DEK path used to decrypt this message's PII fields — not forwarded downstream

All other fields (employeeId, locationId, domain, domainId, eventType, username, userType, eventTimestamp, additionalInfo) already match the sample payload 1:1.
5.3 Outbound payload
* AuditEventPayload.additionalInfo will carry the decrypted emailSentFrom / emailSentTo / smsPhone values (mirrors how displayStatus / everifyCaseEligibility are added today).
* AuditApiRequest gains a new top-level field:


private List<CommunicationPayload> communications;

where CommunicationPayload mirrors Communication but with body already decrypted. This keeps AuditApiClient.postAuditData() unchanged (still posts one generic request object) while giving es-onb-audit-api a distinct, typed list to persist into its own table.


________________


6. New subscriber: CommunicationAuditEventSubscriber (i9-audit-handler)
Thin adapter, mirrors EverifyAuditEventSubscriber exactly:


@Log4j2
@Component
@ConditionalOnProperty(name = "audit.communication-event-subscriber.enabled", havingValue = "true")
public class CommunicationAuditEventSubscriber extends ResilientSubscriber {


    private final CommunicationAuditEventService communicationAuditEventService;


    public CommunicationAuditEventSubscriber(
            @Value("${audit.communication-event-subscriber.name}") String subscriptionName,
            ReactiveSubscriber reactiveSubscriber,
            CommunicationAuditEventService communicationAuditEventService,
            MessageConfirmation messageConfirmation) {
        super(subscriptionName, reactiveSubscriber, messageConfirmation);
        this.communicationAuditEventService = communicationAuditEventService;
    }


    @Override
    public Mono<Void> tryProcessMessage(AcknowledgeablePubsubMessage message) {
        PubsubMessage pubsubMessage = message.getPubsubMessage();
        return communicationAuditEventService.processCommunicationEvent(
            pubsubMessage.getData().toStringUtf8(),
            pubsubMessage.getMessageId());
    }
}

________________


7. New service: CommunicationAuditEventService (i9-audit-handler)
New class (does not touch AuditEventService, since mandatory-field validation differs):


1. Deserialize raw payload → AuditEvent (with communications + gcsBucketPath); set eventId from the PubSub messageId (same convention as today).
2. Decrypt, using PubSubDecryptionService.decrypt(gcsBucketPath, encryptedValue):
   * additionalInfo.emailSentFrom
   * additionalInfo.emailSentTo
   * additionalInfo.smsPhone
   * each communications[i].body
   * Add a small helper decryptIfPresent(String gcsBucketPath, String value) that passes through null/blank values unchanged (avoids unnecessary calls, e.g. the empty SMS subject in the sample payload).
3. Validate only what this domain actually needs: packetId + employeeId (matches AuditEventReferenceRequestDto's real @NotNull constraints downstream — no employeeFactId/employeeRootFactId concept for communication events, unlike Everify).
4. Build AuditEventPayload (with decrypted additionalInfo), AuditEventReferencePayload (packetId, employeeId, i9Id, locationId — no fact-id fields), and the new communications list (decrypted) on AuditApiRequest.
5. Post via the existing AuditApiClient.postAuditData() — no HTTP client changes needed.


________________


8. application.yml additions (i9-audit-handler)
audit:
  everify-event-subscriber:
    name: ${AUDIT_SUBSCRIPTION_ID:es-i9-audit-handler-everify-dev}
    enabled: ${AUDIT_SUBSCRIBER_ENABLED:true}
  communication-event-subscriber:
    name: ${COMMUNICATION_AUDIT_SUBSCRIPTION_ID:es-i9-audit-events-subscriber-dev}
    enabled: ${COMMUNICATION_AUDIT_SUBSCRIBER_ENABLED:true}

No change needed to spring.cloud.gcp.project-id (same project per environment already covers both subscriptions). Per-environment subscription IDs (-dev, -qa, -uat, -prod) are supplied via COMMUNICATION_AUDIT_SUBSCRIPTION_ID in each environment's deployment config, same pattern as the existing AUDIT_SUBSCRIPTION_ID.


________________


9. Downstream changes: es-onb-audit-api (1574_US_ES-ONB-AUDIT-API_WSA)
9.1 New Spanner table: audit_communications
Column
	Type
	Notes
	id
	UUID (PK)
	Generated per row.
	packet_id
	UUID
	FK-style link to audit_event.packet_id.
	event_id
	STRING
	FK-style link to audit_event.event_id.
	channel
	STRING
	EMAIL / SMS.
	subject
	STRING
	Plaintext (not encrypted at source).
	body
	STRING
	Encrypted at rest via CryptographyService, same pattern as AuditEventEntity.username.
	key_store_id
	STRING
	Companion key-store id for body encryption (mirrors AuditEventEntity.keyStoreId).
	created_at
	TIMESTAMP
	
	
	

DDL/migration script itself is out of scope for this repo's plan (managed via the team's Spanner migration tooling) — call out as an infra task in the rollout checklist.
9.2 New entity: AuditCommunicationEntity (es-onb-audit-repository)
Mirrors AuditEventReferenceEntity's structure/annotations (@Table, @PrimaryKey, @Column).
9.3 New repository method
Extend AuditEventRepository (or add AuditCommunicationRepository) with an insert that participates in the same read-write transaction as saveEventAndUpdateReference, e.g.:


public void saveEventReferenceAndCommunications(
        AuditEventReferenceEntity referenceEntity,
        AuditEventEntity eventEntity,
        List<AuditCommunicationEntity> communicationEntities) {
    spannerTemplate.performReadWriteTransaction(tx -> {
        // existing upsert/insert logic for reference + event
        communicationEntities.forEach(tx::insert);
        return null;
    });
}
9.4 New DTO: CommunicationRequestDto (es-onb-audit-dto)
@Data @Builder @NoArgsConstructor @AllArgsConstructor
public class CommunicationRequestDto {
    @NotBlank private String channel;
    private String subject;
    @NotBlank private String body;
}
9.5 AuditCreateRequestDto — add field
@Valid
private List<CommunicationRequestDto> communications;
9.6 AuditDataMapper — add mapping
New toCommunicationEntities(UUID packetId, String eventId, List<CommunicationRequestDto> dtos) method, encrypting body via CryptographyService (same applyUsernameEncryption pattern already used in AuditHistoryServiceImpl) before building each AuditCommunicationEntity.
9.7 AuditHistoryServiceImpl.saveAuditData() — extend
Build the communication entities (encrypting body), then call the extended repository method so reference + event + communications are persisted atomically.
9.8 Controller
No signature change needed — AuditHistoryController.saveAuditData() already accepts the whole AuditCreateRequestDto; the new communications field flows through automatically.


________________


10. Tests to add/update
i9-audit-handler: | Test | Coverage | |---|---| | EverifyAuditEventSubscriberTest (renamed) | Same as today. | | CommunicationAuditEventSubscriberTest | Delegation contract, error propagation (mirrors Everify test). | | CommunicationAuditEventServiceTest | Deserialization, decryption calls (mock PubSubDecryptionService), mandatory-field validation (packetId/employeeId only), decrypted communications + additionalInfo correctness, successful POST. | | Enum tests (AuditDomain, AuditEventType, AuditUserType) | New values serialize/deserialize correctly, case-insensitive matching. |


es-onb-audit-api: | Test | Coverage | |---|---| | AuditDataMapperTest | New toCommunicationEntities() mapping + encryption. | | AuditHistoryServiceImplTest | saveAuditData() persists communications; encryption failure path. | | AuditEventRepositoryTest | New transactional insert method covers reference + event + communications atomically. | | AuditHistoryControllerTest | Request body with communications accepted (schema/validation). |


________________


11. Rollout checklist
* Confirm exact JSON field name for the DEK path on the inbound message with the producer team (assumed gcsBucketPath above).
* Provision the audit_communications Spanner table (DDL via migration tooling) in all environments before enabling the new subscriber in production.
* Provision the PubSub subscription es-i9-audit-events-subscriber-{env} in each GCP project (ews-es-i9-npe-00c4 dev/qa, ews-es-i9-prd-e87d uat/prod); grant the i9-audit-handler service account pubsub.subscriber role.
* Set COMMUNICATION_AUDIT_SUBSCRIPTION_ID env var per environment in deployment configs.
* Keep audit.communication-event-subscriber.enabled=false until the subscription and audit_communications table both exist in that environment.
* Verify pub-sub.decryption.enabled=true and the asymmetric key/version are correctly provisioned per environment before enabling this subscriber against real encrypted traffic.
* Coordinate the es-onb-audit-api deployment (new table + DTO/entity/mapper/repository changes) so it lands before or alongside the new subscriber's rollout.