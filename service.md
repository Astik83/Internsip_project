

```java
public interface VisitorService {

    VisitorResponseDto registerVisitor(VisitorWithVisitRequestDto dto);

    VisitResponseDto addVisit(Integer visitorId, VisitRequestDto dto);

    List<VisitorResponseDto> getAllVisitors();

    VisitorResponseDto getVisitorById(Integer visitorId);

    VisitorResponseDto updateVisitor(Integer visitorId, VisitorRequestDto dto);

    VisitResponseDto updateVisit(Integer visitorId, Integer recordId, VisitRequestDto dto);
}
```

---

## 🛠️ 2. Service Implementation

```java
@Service
public class VisitorServiceImpl implements VisitorService {

    @Autowired
    private VisitorRepository visitorRepository;

    @Autowired
    private VisitRecordRepository visitRecordRepository;

    @Autowired
    private ModelMapper modelMapper;

    // 🔷 1. Register Visitor + First Visit
    @Override
    public VisitorResponseDto registerVisitor(VisitorWithVisitRequestDto dto) {

        // Convert Visitor DTO → Entity
        Visitor visitor = modelMapper.map(dto.getVisitor(), Visitor.class);

        // Generate Unique ID
        visitor.setUniqueId(generateUniqueId());

        visitor.setStatus(VisitorStatus.PENDING);
        visitor.setCreatedAt(LocalDateTime.now());

        // Save Visitor
        Visitor savedVisitor = visitorRepository.save(visitor);

        // Create Visit
        VisitRecord visit = createVisit(dto.getVisit(), savedVisitor);

        // Update status
        savedVisitor.setStatus(VisitorStatus.CHECKED_IN);

        visitRecordRepository.save(visit);

        return mapToVisitorResponse(savedVisitor);
    }

    // 🔷 2. Add New Visit
    @Override
    public VisitResponseDto addVisit(Integer visitorId, VisitRequestDto dto) {

        Visitor visitor = visitorRepository.findById(visitorId)
                .orElseThrow(() -> new RuntimeException("Visitor not found"));

        VisitRecord visit = createVisit(dto, visitor);

        visitor.setStatus(VisitorStatus.CHECKED_IN);

        visitRecordRepository.save(visit);

        return modelMapper.map(visit, VisitResponseDto.class);
    }

    // 🔷 3. Get All Visitors
    @Override
    public List<VisitorResponseDto> getAllVisitors() {

        List<Visitor> visitors = visitorRepository.findAll();

        return visitors.stream()
                .map(this::mapToVisitorResponse)
                .toList();
    }

    // 🔷 4. Get Visitor by ID
    @Override
    public VisitorResponseDto getVisitorById(Integer visitorId) {

        Visitor visitor = visitorRepository.findById(visitorId)
                .orElseThrow(() -> new RuntimeException("Visitor not found"));

        return mapToVisitorResponse(visitor);
    }

    // 🔷 5. Update Visitor
    @Override
    public VisitorResponseDto updateVisitor(Integer visitorId, VisitorRequestDto dto) {

        Visitor visitor = visitorRepository.findById(visitorId)
                .orElseThrow(() -> new RuntimeException("Visitor not found"));

        if (dto.getName() != null) visitor.setName(dto.getName());
        if (dto.getCompany() != null) visitor.setCompany(dto.getCompany());
        if (dto.getContactNumber() != null) visitor.setContactNumber(dto.getContactNumber());
        if (dto.getEmail() != null) visitor.setEmail(dto.getEmail());
        if (dto.getNotes() != null) visitor.setNotes(dto.getNotes());

        visitorRepository.save(visitor);

        return mapToVisitorResponse(visitor);
    }

    // 🔷 6. Update Visit
    @Override
    public VisitResponseDto updateVisit(Integer visitorId, Integer recordId, VisitRequestDto dto) {

        VisitRecord visit = visitRecordRepository.findById(recordId)
                .orElseThrow(() -> new RuntimeException("Visit not found"));

        if (!visit.getVisitor().getVisitorId().equals(visitorId)) {
            throw new RuntimeException("Visit does not belong to this visitor");
        }

        if (dto.getReasonForVisit() != null)
            visit.setReasonForVisit(dto.getReasonForVisit());

        if (dto.getVisitNotes() != null)
            visit.setVisitNotes(dto.getVisitNotes());

        if (dto.getGatePassDuration() != null) {
            visit.setGatePassDuration(dto.getGatePassDuration());
            visit.setGatePassExpiryTime(
                    visit.getEntryTime().plusHours(dto.getGatePassDuration())
            );
        }

        visitRecordRepository.save(visit);

        return modelMapper.map(visit, VisitResponseDto.class);
    }

    // 🔷 COMMON METHODS

    private VisitRecord createVisit(VisitRequestDto dto, Visitor visitor) {

        VisitRecord visit = modelMapper.map(dto, VisitRecord.class);

        LocalDateTime entryTime = LocalDateTime.now();

        visit.setVisitor(visitor);
        visit.setEntryTime(entryTime);

        int duration = dto.getGatePassDuration() != null ? dto.getGatePassDuration() : 1;

        visit.setGatePassDuration(duration);
        visit.setGatePassExpiryTime(entryTime.plusHours(duration));

        visit.setStatusAtTime("CHECKED_IN");

        return visit;
    }

    private String generateUniqueId() {
        return "V" + System.currentTimeMillis(); // simple version
    }

    private VisitorResponseDto mapToVisitorResponse(Visitor visitor) {

        VisitorResponseDto dto = modelMapper.map(visitor, VisitorResponseDto.class);

        if (visitor.getVisitRecords() != null) {
            List<VisitResponseDto> visits = visitor.getVisitRecords()
                    .stream()
                    .map(v -> modelMapper.map(v, VisitResponseDto.class))
                    .toList();

            dto.setVisitRecords(visits);
        }

        return dto;
    }
}
```

---

## ✅ 3. Business Logic Covered

- ✔ Unique ID generation
- ✔ Visitor status update (PENDING → CHECKED_IN → CHECKED_OUT)
- ✔ Visit creation with automatic expiry time calculation
- ✔ Partial updates (null-safe)
- ✔ Relationship handling between Visitor and VisitRecord
- ✔ DTO ↔ Entity mapping using ModelMapper

---

## 🔥 4. Additional Feature: Check-Out API

```java
public void checkoutVisitor(Integer recordId) {

    VisitRecord visit = visitRecordRepository.findById(recordId)
            .orElseThrow(() -> new RuntimeException("Visit not found"));

    visit.setExitTime(LocalDateTime.now());

    Visitor visitor = visit.getVisitor();
    visitor.setStatus(VisitorStatus.CHECKED_OUT);

    visitRecordRepository.save(visit);
}
```

---

## ⏰ 5. Background Expiry Job

Automatically expire visits after gate pass duration.

```java
@Scheduled(fixedRate = 60000) // runs every minute
public void expireVisitors() {

    List<VisitRecord> visits =
            visitRecordRepository.findExpiredVisits(LocalDateTime.now());

    for (VisitRecord visit : visits) {
        visit.getVisitor().setStatus(VisitorStatus.EXPIRED);
    }
}
```

---

