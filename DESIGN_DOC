# 🎬 Project Chronos

**AI Video Orchestration System – System Design Document**

---

## 1. Overview

Project Chronos is a backend system written in **Java (Spring Boot)** that turns a written script into a complete video.

The system solves two big problems most AI video tools have today:

1. **Characters look different in every scene**  
2. **Videos are very short** — most models can only make clips of about 5–10 seconds

Chronos fixes both issues by:

- Creating **one permanent reference image** for the character  
- Breaking the entire script into **strict 5-second scenes**  
- Generating video and audio for each scene (in parallel)  
- Stitching everything together cleanly using **FFmpeg**

The result: a long, consistent-looking video made fully automatically.

The system leverages **Spring AI** for model-agnostic integration with various AI providers, enabling seamless abstraction and switching between LLMs, image generation, video generation, TTS, and music models.

---

## 2. Core Idea: Consistency and Chunking

### 2.1 Keeping the Character Looking the Same

The biggest problem with AI video right now is that the character changes face, clothes, age, or style between scenes.

Chronos prevents this by using **one single reference image** for the whole video.

How it works:

1. An LLM writes a very detailed description of the character (face, hair, clothes, age, style, etc.)  
2. An image generation model creates **one high-quality portrait** from that description  
3. This portrait is sent as a reference image to every single video generation request

This technique is called **Image-to-Video (I2V)** and is widely used in serious AI video pipelines.

### 2.2 Breaking the Script into 5-Second Pieces

AI video models work best with short clips. So we force every scene to be **exactly around 5 seconds**.

We use this simple timing rule:

- Average speaking speed ≈ **2.4 words per second**  
- → **5 seconds ≈ 12 words**

Every narration text is checked and adjusted so it fits close to 12 words. This keeps audio and video naturally in sync without weird stretching.

---

## 3. Which AI Models We Can Use (2025–2026)

| Stage                        | Purpose                              | Best / Recommended Models right now               | Strong Alternatives                                 | Main Trade-offs / Notes                                  |
|------------------------------|--------------------------------------|---------------------------------------------------|-----------------------------------------------------|----------------------------------------------------------|
| Character Description        | Write detailed character prompt      | Claude 3.5 Sonnet, Claude 3.7, GPT-4o, Gemini 2.0 | Grok-3, Llama 4, Qwen 2.5-Max, DeepSeek-R1          | Needs very good detail and consistency                   |
| Character Portrait           | Create one high-quality reference    | Flux.1 [pro], Midjourney v6.1 / v7, Ideogram 2.0  | SD3.5 Large, Recraft v3, Imagen 4, Lumina-Image     | Quality & following the prompt closely is most important |
| Video Generation (I2V)       | Turn reference image + prompt → 5s clip | Runway Gen-4 / Gen-4 Turbo, Kling 2.0 / 2.1     | Luma Dream Machine 1.6+, Hailuo MiniMax, Vidu       | Kling and Runway currently best at keeping character consistent |
| Text-to-Speech (Narration)   | Create natural voice-over            | ElevenLabs v3 / Flash v2.5, Cartesia Sonic        | PlayHT v3, XTTS-v2 fine-tuned, Sesame CSM-1B        | ElevenLabs = highest quality, Cartesia = fastest         |
| Background Music (optional)  | Add cinematic / mood music           | Suno v4, Udio v2, Stable Audio 2.0                | Mubert, AIVA, MusicGen fine-tuned                   | Can be one track for whole video or per scene            |
| Video Upscaling / Polish     | Make final video sharper (optional)  | Topaz Video AI, AVCLabs, HitPaw                   | SUPIR, Real-ESRGAN variants                         | Only needed if base model output is low resolution       |

The system is built to be **model-agnostic** — we can swap providers easily per project or even per scene using Spring AI abstractions for LLMs, embeddings, image generation, and other AI capabilities.

---

## 4. Data Model

| Entity    | Field              | Type | What it means                              |
|-----------|--------------------|------|--------------------------------------------|
| Project   | id                 | UUID | Unique ID for the whole video project      |
| Project   | status             | ENUM | CREATED, CHARACTER_APPROVAL, PROCESSING, DONE, FAILED |
| Project   | final_video_url    | TEXT | Link to the finished video                 |
| Character | master_prompt      | TEXT | Very detailed character description        |
| Character | anchor_img_url     | TEXT | URL of the one master reference image      |
| Segment   | order_index        | INT  | Scene number (1, 2, 3…)                    |
| Segment   | action_prompt      | TEXT | What should happen visually in this scene  |
| Segment   | narration          | TEXT | The spoken text (~12 words)                |
| Segment   | video_url          | TEXT | Generated 5-second video clip              |
| Segment   | audio_url          | TEXT | Generated narration audio                  |
| Segment   | status             | ENUM | PENDING, QUEUED, GENERATING, COMPLETED, FAILED     |

---

## 5. User Stories

- **US-1: Character Preview**  
  As a user, I want to see and approve the character image before the system starts making the video.

- **US-2: Perfectly Timed Scenes**  
  As a user, I want the script split into scenes that exactly match video length.

- **US-3: Faster Processing**  
  As the system, I want to create video and audio at the same time to finish quicker.

- **US-4: One Final Video**  
  As a user, I want one clean MP4 file with all scenes, voice, and background music.

- **US-5: Restart Resiliency**  
  As the system, I want to recover from restarts by persisting queues and re-queuing unfinished work.

---

## 6. End-to-End Workflow

1. **User sends script** + style  
2. **Create character**  
   - LLM writes detailed description  
   - Image model makes one master portrait  
   - (Optional: user approves the image)  
3. **Split script into scenes**  
   - Each scene ≈ 12 words narration  
   - Each scene gets visual prompt + narration  
4. **Generate scenes in parallel**  
   - Video: send reference image + action prompt → 5s clip  
   - Audio: send narration text → voice clip  
   - Fix timing (add silence or tiny speed change)  
5. **Assemble final video with FFmpeg**  
   - All clips resized to 1920×1080 @ 30fps  
   - All clips joined in order  
   - Voice + background music mixed  
6. **Return link** to the finished video

In case of restarts, a recovery process scans for incomplete projects and re-queues pending segments.

---

## 7. API – Main Endpoint

```http
POST /api/v1/projects
Content-Type: application/json

{
  "script": "Once upon a time, a young entrepreneur named Alex had a great idea...",
  "style": "corporate",
  "voice": "male-professional", 
  "music": "uplifting-cinematic"
}
```

**Response:**
```json
{
  "projectId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "CREATED",
  "pollUrl": "/api/v1/projects/550e8400-e29b-41d4-a716-446655440000"
}
```

### Full API Spec

| Endpoint | Method | Description | Auth |
|----------|--------|-------------|------|
| `POST /api/v1/projects` | Create new project from script | Returns projectId for polling | API Key |
| `GET /api/v1/projects/{id}` | Get project status + progress | Returns current status, character preview, segment progress | API Key |
| `POST /api/v1/projects/{id}/approve-character` | Approve character image | Triggers script splitting | API Key |
| `GET /api/v1/projects/{id}/final-video` | Get download URL | 302 redirect to S3 | Public |
| `DELETE /api/v1/projects/{id}` | Cleanup project | Deletes all assets after 7 days auto | API Key |

**WebSocket (Optional):** `ws://api.chronos.com/ws/projects/{id}` for real-time updates.

---

## 8. System Architecture

**Tech Stack:**
- **Backend:** Java 21 + Spring Boot 3.3
- **AI Integration:** Spring AI (for LLM, image, video, TTS, and music model abstractions)
- **Database:** PostgreSQL 16 (JPA/Hibernate)
- **Queue:** Redis 7 (for parallel segment processing, with AOF persistence enabled)
- **Storage:** AWS S3 (videos/assets)
- **Processing:** Docker + FFmpeg 6
- **Monitoring:** Prometheus + Grafana
- **API Gateway:** Spring Cloud Gateway (future scaling)

**High-Level Diagram:**

```
┌─────────────┐    ┌──────────────┐    ┌──────────────┐
│    User     │───▶│   API Layer  │───▶│   WebSocket  │
│             │    │ (Spring Boot)│    │   (Optional) │
└─────────────┘    └──────────────┘    └──────────────┘
                         │
                         ▼
    ┌─────────────────────────────────────────────────────────────┐
    │                        Core Services                        │
    │  ├── Spring AI (LLM/Image/Video/TTS/Music)                  │
    │  ├── Script Splitter                                        │
    │  ├── Segment Processor ◄──┐                                 │
    │  ├── Recovery Job    ─────┼─── Redis Queue (AOF) ───────────┤
    │  └── FFmpeg Assembler ────┘                                 │
    └─────────────────────────────────────────────────────────────┘
                                │
                                ▼
                    ┌─────────────────┐    ┌──────────┐
                    │   PostgreSQL    │───▶│   S3     │
                    │ (Projects/DB)   │    │ (Videos) │
                    └─────────────────┘    └──────────┘
```

**Deployment:** Docker Compose for dev, Kubernetes for prod.

**Restart Resiliency:**
- Redis configured with AOF persistence to survive restarts.
- A startup recovery job scans PostgreSQL for incomplete segments and re-queues them.
- Idempotency ensured via segment status checks to avoid duplicates.

---

## 9. Detailed Service Implementation

### 9.1 Character Generation Service
```java
@Service
public class CharacterService {
    @Autowired
    private ChatClient chatClient; // Spring AI for LLM

    @Autowired
    private ImageClient imageClient; // Spring AI for image gen

    public CompletableFuture<Character> generateAsync(Project project) {
        // 1. LLM → detailed prompt (using Spring AI)
        Prompt prompt = new Prompt("Generate detailed character description from script: " + project.getScript());
        ChatResponse response = chatClient.call(prompt);
        String masterPrompt = response.getResult().getOutput().getContent();
        
        // 2. Image gen → anchor image (using Spring AI)
        ImageOptions options = ImageOptions.builder().withModel("flux.1-pro").build();
        ImageResponse imgResponse = imageClient.call(new ImagePrompt(masterPrompt, options));
        String anchorImgUrl = imgResponse.getResult().getOutput().getUrl();
        
        // 3. Save & notify
        Character character = new Character(masterPrompt, anchorImgUrl);
        project.setCharacter(character);
        return CompletableFuture.completedFuture(character);
    }
}
```

### 9.2 Script Splitter
**Algorithm:** Greedy word-chunking to ~12 words/scene.
```java
public List<Segment> splitScript(String fullScript, Character character) {
    String[] words = fullScript.split("\\s+");
    List<Segment> segments = new ArrayList<>();
    
    for (int i = 0; i < words.length; i += 12) {
        String narration = String.join(" ", Arrays.copyOfRange(words, i, Math.min(i+12, words.length)));
        // Use Spring AI for visual prompt generation
        Prompt prompt = new Prompt("Generate visual prompt for narration: " + narration + " with character: " + character.getMasterPrompt());
        ChatResponse response = chatClient.call(prompt);
        String actionPrompt = response.getResult().getOutput().getContent();
        
        segments.add(new Segment(orderIndex++, actionPrompt, narration));
    }
    return segments;
}
```

### 9.3 Parallel Segment Processing
```java
@Service
public class SegmentProcessor {
    @Async
    public void processProjectSegments(UUID projectId) {
        List<Segment> pending = segmentRepo.findPendingByProject(projectId);
        
        pending.parallelStream().forEach(seg -> {
            // Use Spring AI for video and TTS
            videoClient.generateAsync(seg);  // I2V with character anchor
            ttsClient.generateAsync(seg);    // Narration audio
        });
    }
}
```

### 9.4 FFmpeg Assembly
```bash
# Final video concat + audio mix (executed via ProcessBuilder)
ffmpeg -i video1.mp4 -i audio1.wav -i bgm.mp3 \
  -filter_complex "[0:v]scale=1920:1080,fps=30[v0];[v0][1:a][2:a]concat=n=1:v=1:a=1[outv][outa]" \
  -map "[outv]" -map "[outa]" final.mp4
```

### 9.5 Recovery Job for Restarts
```java
@Component
public class RecoveryJob implements ApplicationRunner {
    @Autowired
    private SegmentRepository segmentRepo;
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @Override
    public void run(ApplicationArguments args) {
        List<Project> processingProjects = projectRepo.findByStatus("PROCESSING");
        for (Project project : processingProjects) {
            List<Segment> incomplete = segmentRepo.findIncompleteByProject(project.getId());
            incomplete.forEach(seg -> {
                if (seg.getStatus() != "COMPLETED") {
                    seg.setStatus("QUEUED");
                    redisTemplate.opsForList().rightPush("segment-queue:" + project.getId(), seg);
                }
            });
        }
    }
}
```

---

## 10. Configuration & Model Abstraction

**application.yml:**
```yaml
chronos:
  llm:
    provider: claude
    api-key: ${LLM_API_KEY}
    model: claude-3-5-sonnet-20241022
  
  video:
    provider: runway
    model: gen4-turbo
    api-key: ${RUNWAY_API_KEY}
  
  tts:
    provider: elevenlabs
    voice-id: "pNInz6obpgDQGcFmaJgB"  # Adam
  
  s3:
    bucket: chronos-videos-${ENV}

spring:
  ai:
    openai:  # Example for OpenAI integration
      api-key: ${OPENAI_API_KEY}
    anthropic:  # For Claude
      api-key: ${ANTHROPIC_API_KEY}
```

**Model Switching:** Change `provider` env var → Spring AI routes to different clients.

**Redis Persistence:**
```yaml
spring:
  data:
    redis:
      appendonly: yes  # Enable AOF persistence
      aof-fsync-everysec: yes
```

---

## 11. Error Handling & Retries

| Error Type | Action | Max Retries |
|------------|--------|-------------|
| Provider timeout | Exponential backoff | 3 |
| Low-quality video | Regenerate with prompt tweak | 2 |
| Audio/video desync | Add silence pad | Auto |
| Assembly failure | Fallback to simple concat | 1 |
| Restart/Queue loss | Recovery job re-queues | Auto |

**Dead Letter Queue:** Failed segments → Redis `dlq:failed` for manual review.

---

## 12. Performance & Scaling

- **5-min script** (25 scenes) → **~15 mins** total (parallel gen)
- **Cost:** ~$2-5 per minute of final video
- **Horizontal Scale:** Multiple `SegmentProcessor` instances pull from shared Redis queue
- **Rate Limits:** Provider-specific throttling (Runway: 10/min)

**Monitoring Metrics:**
- `segments_processing_duration_seconds`
- `project_completion_rate`
- `provider_error_rate`
- `recovery_job_executions`

---

## 13. Developer Setup

```bash
# Clone & run
git clone <repo>
docker-compose up postgres redis
mvn spring-boot:run

# Test
curl -X POST http://localhost:8080/api/v1/projects \
  -H "Content-Type: application/json" \
  -d '{"script": "Your test script here..."}'
```
