# Hermes Agent

_Tài liệu tổng hợp từ 3 PDF trong thư mục `1/`, dùng để hiểu Hermes Agent từ khái niệm, cơ chế hoạt động đến ví dụ code dự án nhỏ._

---

## 📋 Tổng quan

Hermes Agent là một framework AI Agent cụ thể của Nous Research. Nếu chatbot truyền thống chủ yếu nhận câu hỏi rồi trả lời bằng văn bản, Hermes hướng tới mô hình "giao việc": hiểu mục tiêu, lập kế hoạch, dùng công cụ, ghi nhớ thông tin, chạy nhiệm vụ định kỳ và học lại quy trình từ kinh nghiệm.

Điểm cần nhớ: Hermes không chỉ là một prompt hay chatbot có thêm tool. Nó là một agent runtime có nhiều lớp vận hành: `Memory`, `Skills`, `Tools`, `Cron`, `Gateway` và `Profile`. Các lớp này giúp Hermes làm việc lâu dài trong repo, môi trường local/cloud, terminal, nền tảng nhắn tin và các workflow lặp lại.

| Nội dung | AI Agent nói chung | Hermes Agent |
| --- | --- | --- |
| **Bản chất** | Khái niệm hệ thống AI có khả năng hành động | Framework/sản phẩm AI Agent cụ thể |
| **Chức năng chính** | Lập kế hoạch, dùng tool, xử lý nhiệm vụ nhiều bước | Có thêm memory, skills, subagents, cron, terminal backend, messaging gateway |
| **Bộ nhớ** | Tùy hệ thống, có thể có hoặc không | Có persistent memory và session search |
| **Kỹ năng** | Không mặc định | Có hệ thống skills, có thể tạo và cải thiện skill |
| **Làm việc với dự án** | Tùy thiết kế | Có thể làm việc với terminal, file, repo, môi trường local/cloud |
| **Tự cải thiện** | Không mặc định | Được thiết kế quanh learning loop |

> 📌 **Tóm tắt:** AI Agent là khái niệm chung. Hermes Agent là một triển khai cụ thể, có bộ nhớ, kỹ năng, công cụ, lịch chạy, gateway và profile để làm việc lâu dài hơn.

---

## ⚙️ Kiến trúc vận hành

Hermes hoạt động như một hệ thống nhiều lớp. Ở giữa là `AIAgent`, vòng lặp điều phối chịu trách nhiệm chọn provider/model, dựng system prompt, đưa schema tool cho model, nhận tool call, chạy tool, retry/fallback, nén context và lưu session. Xung quanh vòng lặp này là 6 thành phần chính: profile xác định môi trường làm việc; memory cung cấp thông tin đã nhớ; skills cung cấp quy trình đã học; tools giúp agent hành động; cron giúp chạy nhiệm vụ theo lịch; gateway giúp agent giao tiếp qua nhiều nền tảng.

```mermaid
flowchart LR
    accTitle: Hermes Agent Components
    accDescr: Diagram showing the main Hermes Agent components and the role each component plays in long-running agent work

    hermes([🤖 Hermes Agent])

    memory[💾 Memory]
    skills[📚 Skills]
    tools[🔧 Tools]
    cron[⏰ Cron]
    gateway[🌐 Gateway]
    profile[👤 Profile]

    memory_role[Nhớ thông tin quan trọng]
    skills_role[Nhớ quy trình làm việc]
    tools_role[Hành động trong môi trường thật]
    cron_role[Tự chạy nhiệm vụ theo lịch]
    gateway_role[Kết nối nhiều nền tảng]
    profile_role[Tách agent theo mục đích]

    hermes --> memory --> memory_role
    hermes --> skills --> skills_role
    hermes --> tools --> tools_role
    hermes --> cron --> cron_role
    hermes --> gateway --> gateway_role
    hermes --> profile --> profile_role

    classDef core fill:#dbeafe,stroke:#2563eb,stroke-width:2px,color:#1e3a5f
    classDef support fill:#dcfce7,stroke:#16a34a,stroke-width:2px,color:#14532d
    classDef output fill:#f3f4f6,stroke:#6b7280,stroke-width:2px,color:#1f2937

    class hermes core
    class memory,skills,tools,cron,gateway,profile support
    class memory_role,skills_role,tools_role,cron_role,gateway_role,profile_role output
```

### Các thành phần chính

| Thành phần | Vai trò | Cơ chế thực tế |
| --- | --- | --- |
| **Memory** | Ghi nhớ thông tin quan trọng về người dùng, dự án, lỗi, môi trường | Dùng bounded curated memory: `MEMORY.md` và `USER.md` được giới hạn dung lượng, inject vào system prompt đầu phiên; lịch sử dài nằm trong SQLite `state.db` và tìm lại bằng session search/FTS5 khi cần |
| **Skills** | Lưu quy trình làm việc thành hướng dẫn có thể tái sử dụng | Lưu trong `~/.hermes/skills/` dạng `SKILL.md`, dùng progressive disclosure: chỉ xem danh sách/tóm tắt trước, khi khớp task mới `skill_view()` để nạp toàn bộ; skill tự sinh nên là draft và qua human review/curator trước khi dùng rộng |
| **Tools** | Cho phép agent đọc/sửa file, chạy terminal, tìm web, dùng browser | Model chỉ tạo `tool_call`; Hermes runtime mới dispatch thật qua tool registry. Một số tool đặc biệt như todo, memory, session search, delegate task được agent loop xử lý trực tiếp vì cần state cấp agent |
| **Cron** | Chạy tác vụ định kỳ mà không cần nhắc lại | Gateway daemon tick theo chu kỳ, đọc `~/.hermes/cron/jobs.json`, tạo fresh `AIAgent` session, inject skill nếu job yêu cầu, gửi output về target và cập nhật `next_run_at`; dùng lock để tránh tick trùng |
| **Gateway** | Kết nối Hermes với Telegram, Discord, Slack, Email và tool gateway | Nhận tin từ nền tảng ngoài, chuyển vào `AIAgent`, áp dụng toolset phù hợp với từng platform, rồi gửi kết quả về đúng kênh gốc |
| **Profile** | Tách nhiều agent theo mục đích, mỗi profile có memory/skills/session riêng | Mỗi profile có cấu hình, memory, skills, session và gateway state riêng; khi dựng prompt, Hermes còn đọc context files như `SOUL.md`, `AGENTS.md`, `CLAUDE.md`, `.cursorrules` theo priority và phát hiện thêm context ở thư mục con khi cần |

---

## 💾 Memory: ghi nhớ dài hạn

Memory là bộ nhớ dài hạn của Hermes. Nó không lưu toàn bộ lịch sử chat, mà chỉ lưu thông tin ngắn gọn, được chọn lọc và có giá trị sử dụng lại. Cách này giúp agent nhớ đúng thứ cần nhớ, tránh làm prompt quá tải hoặc gây nhiễu.

```mermaid
flowchart TD
    accTitle: Hermes Memory Flow
    accDescr: Flow showing how Hermes decides whether information should be saved into memory and how it is reused in later sessions

    user_work([👤 Người dùng làm việc]) --> observe[🔍 Hermes quan sát phiên làm việc]
    observe --> important{Thông tin quan trọng?}
    important -->|Không| skip[Không lưu]
    important -->|Có| classify{Thuộc loại nào?}
    classify -->|Nhân cách, nguyên tắc, profile| soul_memory[🧭 Lưu vào SOUL.md]
    classify -->|Dự án, lỗi, workflow| project_memory[💾 Lưu vào MEMORY.md]
    classify -->|Sở thích, giao tiếp| user_memory[👤 Lưu vào USER.md]
    soul_memory --> next_session[Phiên làm việc sau]
    project_memory --> next_session[Phiên làm việc sau]
    user_memory --> next_session
    next_session --> prompt_builder[prompt_builder.py tổng hợp SOUL + MEMORY + USER]
    prompt_builder --> prompt[Đưa vào system prompt]
    prompt --> better[✅ Xử lý phù hợp hơn]

    classDef decision fill:#fef9c3,stroke:#ca8a04,stroke-width:2px,color:#713f12
    classDef data fill:#dcfce7,stroke:#16a34a,stroke-width:2px,color:#14532d
    classDef process fill:#dbeafe,stroke:#2563eb,stroke-width:2px,color:#1e3a5f

    class important,classify decision
    class soul_memory,project_memory,user_memory data
    class observe,prompt_builder,prompt,better process
```

### Ví dụ rõ ràng

Nếu người dùng thường làm backend bằng `FastAPI` và database `SQLite`, Hermes có thể ghi nhớ thông tin đó. Lần sau khi người dùng yêu cầu tạo backend, agent sẽ ưu tiên stack quen thuộc thay vì hỏi lại từ đầu.

| File Markdown | Vai trò | Ví dụ |
| --- | --- | --- |
| `SOUL.md` | Nhân cách lõi, giọng văn, nguyên tắc ứng xử, giới hạn kỷ luật của agent | "Luôn kiểm tra repo trước khi sửa code" |
| `MEMORY.md` | Thông tin dự án, lỗi đã gặp, workflow, môi trường | "Project dùng FastAPI + SQLite" |
| `USER.md` | Sở thích người dùng, cách giao tiếp, format trả lời | "Người dùng thích giải thích tiếng Việt ngắn gọn" |

Trong luồng này, `SOUL.md` không phải một phần phụ tách riêng bên dưới, mà là một nhánh cùng cấp với `MEMORY.md` và `USER.md`. Khi bắt đầu phiên mới, `prompt_builder.py` tổng hợp cả ba file vào system prompt: `SOUL.md` trả lời "agent là kiểu cộng sự nào?", `MEMORY.md` trả lời "agent đang làm việc trong môi trường nào?", còn `USER.md` trả lời "agent đang phục vụ ai?".

Vì vậy, nếu chỉ mô tả `MEMORY.md` và `USER.md`, bức tranh về kiểm soát hành vi dài hạn của Hermes sẽ thiếu một chân kiềng quan trọng. Với từng profile hoặc subagent, `SOUL.md` có thể khác nhau để tạo ra hành vi chuyên biệt: coder agent cần kỷ luật kiểm thử và đọc code trước khi sửa; research agent cần thói quen trích nguồn và phân biệt giả định; personal assistant cần ưu tiên ngữ cảnh người dùng và cách giao tiếp quen thuộc.

### Cơ chế hoạt động

`MEMORY.md` lưu ghi chú của agent về môi trường, project, workflow, lỗi đã gặp và bài học đã học. `USER.md` lưu thông tin về người dùng như preference, cách giao tiếp, thói quen làm việc và format trả lời quen thuộc. Hai file này không được để phình vô hạn: `MEMORY.md` thường bị giới hạn khoảng 2.200 ký tự, còn `USER.md` khoảng 1.375 ký tự, rồi được inject vào system prompt khi session bắt đầu.

Thiết kế này có ba ý quan trọng. Một là memory được đưa vào prompt ở đầu phiên, nên model luôn thấy những thông tin quan trọng nhất mà không cần tìm lại. Hai là memory có giới hạn ký tự; khi đầy, tool có thể trả lỗi và agent phải tự gộp, nén hoặc xóa thông tin cũ, tránh tình trạng "memory càng lâu càng loạn". Ba là memory không thay đổi ngay trong system prompt giữa phiên: nếu agent lưu memory mới, nội dung được ghi xuống file ngay, nhưng thường chỉ xuất hiện trong prompt từ session sau. Cách này giữ prompt ổn định và tận dụng prompt caching tốt hơn.

Ngoài memory ngắn, Hermes còn có session search. Toàn bộ session CLI hoặc messaging có thể được lưu vào SQLite `~/.hermes/state.db` và tìm bằng FTS5 full-text search. Khi cần nhớ một việc đã nói từ vài tuần trước, agent không nhét toàn bộ lịch sử vào prompt, mà gọi `session_search`, lấy đoạn liên quan, rồi dùng đoạn đó trong câu trả lời.

---

## 📚 Skills: học và tái sử dụng quy trình

Skills giúp Hermes biến kinh nghiệm thành quy trình có thể dùng lại. Nếu memory nhớ thông tin, skills nhớ cách làm. Một skill thường là file hướng dẫn như `SKILL.md`, chỉ được tải khi nhiệm vụ liên quan.

### Cơ chế hoạt động

Skills thường được lưu ở `~/.hermes/skills/`, mỗi skill là một thư mục hoặc tài liệu có `SKILL.md` chứa metadata, mô tả khi nào dùng, quy trình thao tác, ràng buộc và ví dụ. Điểm quan trọng là Hermes không nạp toàn bộ nội dung skills vào prompt ngay từ đầu. Nó dùng progressive disclosure để tiết kiệm token:

| Cấp | Cơ chế | Nội dung agent thấy |
| --- | --- | --- |
| Level 0 | `skills_list()` | Tên skill và mô tả ngắn |
| Level 1 | `skill_view(name)` | Toàn bộ `SKILL.md` |
| Level 2 | `skill_view(name, path)` | File phụ như template, script, reference |

Nhờ vậy, agent chỉ load skill đầy đủ khi task thật sự khớp. Khi hoàn thành một nhiệm vụ phức tạp, gặp lỗi rồi tìm ra cách đúng, hoặc được người dùng sửa cách làm, Hermes có thể dùng `skill_manage` để tạo, patch, edit, delete, `write_file` hoặc `remove_file` trong skill. Nhưng skill tự sinh ban đầu nên là draft; curator hoặc con người cần review để tránh lưu nhầm quy trình vá tạm thành kinh nghiệm chính thức.

```mermaid
flowchart TD
    accTitle: Hermes Skill Reuse
    accDescr: Flow showing how Hermes checks for a matching skill, reuses it when available, or creates a new skill after solving a repeatable task

    task([👤 Người dùng giao nhiệm vụ]) --> analyze[Hermes phân tích yêu cầu]
    analyze --> check_skill[Kiểm tra danh sách skills]
    check_skill --> has_skill{Có skill phù hợp?}
    has_skill -->|Có| load_skill[📚 Load SKILL.md]
    load_skill --> follow_process[Làm theo quy trình đã lưu]
    follow_process --> stable[✅ Nhanh và ổn định hơn]
    has_skill -->|Không| solve_from_scratch[Hermes tự xử lý từ đầu]
    solve_from_scratch --> reusable{Quy trình tái sử dụng được?}
    reusable -->|Có| create_skill[Tạo draft SKILL.md]
    reusable -->|Không| finish_no_skill[Kết thúc không tạo skill]
    create_skill --> review{Con người review?}
    review -->|Đạt| approve[Approve skill]
    review -->|Chưa đạt| revise[Sửa hoặc bỏ draft]
    approve --> next_time[Lần sau tái sử dụng]

    classDef decision fill:#fef9c3,stroke:#ca8a04,stroke-width:2px,color:#713f12
    classDef success fill:#dcfce7,stroke:#16a34a,stroke-width:2px,color:#14532d
    classDef process fill:#dbeafe,stroke:#2563eb,stroke-width:2px,color:#1e3a5f

    class has_skill,reusable,review decision
    class stable,approve,next_time success
    class analyze,check_skill,load_skill,follow_process,solve_from_scratch,create_skill,revise process
```

### Ví dụ skill sửa lỗi React

Khi Hermes nhiều lần sửa lỗi build trong dự án React, nó có thể rút ra quy trình:

1. Chạy `npm run build`
2. Đọc error log
3. Tìm file gây lỗi
4. Sửa `import`, `type` hoặc `component`
5. Chạy lại build
6. Nếu còn lỗi thì lặp lại

Sau đó Hermes có thể lưu thành bản nháp skill. Khi con người review và approve, lần sau gặp lỗi build React, agent tải skill đó thay vì mò lại từ đầu.

---

## 🔧 Tools: biến suy nghĩ thành hành động

LLM không tự chạy lệnh. Nó chỉ quyết định nên gọi công cụ nào. Hermes runtime nhận `tool_call`, chọn tool phù hợp, thực thi hành động thật rồi trả kết quả lại cho mô hình.

### Cơ chế hoạt động

Tools là các hàm thật mà agent có thể gọi, ví dụ web search, terminal, file edit, browser, memory, session search, cron, delegation hoặc MCP integrations. Hermes tổ chức tools thành các toolset để bật/tắt theo môi trường. CLI có thể cho phép terminal và file tool rộng hơn; Telegram, Discord hoặc Slack có thể bị giới hạn để tránh hành động nguy hiểm từ kênh chat.

Luồng dispatch thường đi như sau:

```text
Model tạo tool_call
   ↓
run_agent.py nhận tool_call
   ↓
model_tools.handle_function_call()
   ↓
Nếu là tool đặc biệt: todo / memory / session_search / delegate_task
   → agent loop xử lý trực tiếp vì cần state cấp agent
Nếu là tool thường:
   → registry.dispatch()
   → chạy handler thật
   → trả JSON/string result về model
```

Điểm cần nhớ: model chỉ đề xuất gọi tool theo schema. Phần runtime mới là nơi thực sự đọc file, sửa file, chạy command, mở browser hoặc lưu memory. Vì vậy, tool system là lớp biến suy nghĩ của LLM thành hành động có kiểm soát.

```mermaid
flowchart TD
    accTitle: Hermes Tool Call Loop
    accDescr: Flow showing how Hermes decides whether a task needs external action, dispatches tool calls, observes results, and loops until the task is complete

    request([👤 Người dùng giao nhiệm vụ]) --> llm[🧠 LLM phân tích yêu cầu]
    llm --> needs_action{Cần hành động bên ngoài?}
    needs_action -->|Không| direct_answer[Trả lời trực tiếp]
    needs_action -->|Có| tool_call[LLM tạo tool_call]
    tool_call --> runtime[⚙️ Hermes runtime nhận tool_call]
    runtime --> choose_tool{Chọn loại tool}
    choose_tool -->|File| file_tool[Đọc hoặc sửa file]
    choose_tool -->|Terminal| terminal_tool[Chạy lệnh, test, build]
    choose_tool -->|Web| web_tool[Tìm kiếm thông tin]
    choose_tool -->|Browser| browser_tool[Mở web, click, nhập dữ liệu]
    file_tool --> observation[📥 Kết quả trả về LLM]
    terminal_tool --> observation
    web_tool --> observation
    browser_tool --> observation
    observation --> done{Nhiệm vụ xong?}
    done -->|Chưa| llm
    done -->|Rồi| final[✅ Trả kết quả cuối]

    classDef decision fill:#fef9c3,stroke:#ca8a04,stroke-width:2px,color:#713f12
    classDef action fill:#dbeafe,stroke:#2563eb,stroke-width:2px,color:#1e3a5f
    classDef success fill:#dcfce7,stroke:#16a34a,stroke-width:2px,color:#14532d

    class needs_action,choose_tool,done decision
    class llm,tool_call,runtime,file_tool,terminal_tool,web_tool,browser_tool,observation action
    class final success
```

### Ví dụ sửa lỗi backend

Lỗi:

```text
ModuleNotFoundError: No module named 'routers'
```

Hermes có thể xử lý theo chuỗi:

1. Chạy app và nhận lỗi từ terminal
2. Đọc `backend/main.py`
3. Thấy dòng `from routers import transactions`
4. Kiểm tra thư mục `backend`
5. Phát hiện chưa có thư mục `routers`
6. Tạo `backend/routers/`
7. Tạo `backend/routers/__init__.py`
8. Tạo `backend/routers/transactions.py`
9. Chạy lại `uvicorn` để kiểm tra

Điểm quan trọng là Hermes sửa dựa trên phản hồi thật từ terminal, không chỉ đoán theo cảm tính.

---

## ⏰ Cron: tự động chạy nhiệm vụ theo lịch

Cron giúp Hermes thực hiện nhiệm vụ định kỳ mà không cần người dùng nhắc lại. Ví dụ: "Mỗi sáng lúc 9 giờ, hãy tìm tin tức AI mới và gửi tóm tắt cho tôi."

### Cơ chế hoạt động

Cron chạy trong gateway daemon. Job có thể là tác vụ một lần hoặc lặp lại, có prompt, lịch chạy, target gửi kết quả và có thể gắn skill vào job. Cơ chế chạy thường là:

```text
Gateway daemon tick mỗi 60 giây
   ↓
Đọc ~/.hermes/cron/jobs.json
   ↓
Kiểm tra job nào đến giờ chạy
   ↓
Tạo fresh AIAgent session
   ↓
Inject skill nếu job có skill
   ↓
Chạy prompt tới khi hoàn thành
   ↓
Gửi output về target
   ↓
Cập nhật next_run_at
```

Để tránh scheduler bị chạy trùng, Hermes có thể dùng lock file như `~/.hermes/cron/.tick.lock`. Cron session cũng nên bị giới hạn quyền tạo cron job mới bên trong cron, nếu không agent có thể vô tình tạo vòng lặp scheduling ngoài ý muốn.

```mermaid
flowchart TD
    accTitle: Hermes Cron Job Flow
    accDescr: Flow showing how Hermes stores a recurring job, checks the schedule, creates an isolated session, runs the task, and sends the result

    create_job([👤 Tạo tác vụ định kỳ]) --> save_job[⏰ Lưu thời gian, prompt, kênh gửi]
    save_job --> daemon[Gateway daemon kiểm tra định kỳ]
    daemon --> due{Đến giờ chạy?}
    due -->|Chưa| daemon
    due -->|Rồi| new_session[Tạo agent session mới]
    new_session --> load_prompt[Load prompt đã lưu]
    load_prompt --> load_context[Load memory và skill nếu cần]
    load_context --> run_task[Hermes thực hiện nhiệm vụ]
    run_task --> send_result[📤 Gửi kết quả về kênh đã chọn]
    send_result --> update_next[Cập nhật lần chạy tiếp theo]
    update_next --> daemon

    classDef decision fill:#fef9c3,stroke:#ca8a04,stroke-width:2px,color:#713f12
    classDef process fill:#dbeafe,stroke:#2563eb,stroke-width:2px,color:#1e3a5f
    classDef success fill:#dcfce7,stroke:#16a34a,stroke-width:2px,color:#14532d

    class due decision
    class save_job,daemon,new_session,load_prompt,load_context,run_task,update_next process
    class send_result success
```

### Ví dụ tác vụ định kỳ

| Lịch | Tác vụ |
| --- | --- |
| Mỗi sáng 8h | Gửi bản tin AI |
| Mỗi tối 9h | Tổng kết công việc trong ngày |
| Mỗi thứ Hai | Kiểm tra issue mới trong GitHub |
| Mỗi tuần | Tạo báo cáo tiến độ học tập |

---

## 🌐 Gateway: kết nối nhiều nền tảng

Gateway là lớp trung gian giúp Hermes nhận tin nhắn từ nền tảng bên ngoài, chuyển vào Hermes xử lý, rồi gửi kết quả trả lại đúng nơi người dùng đã gửi yêu cầu.

### Cơ chế hoạt động

Messaging gateway nhận event từ Telegram, Discord, Slack, WhatsApp hoặc Email, chuẩn hóa thành request cho `AIAgent`, rồi gửi câu trả lời về đúng thread, chat hoặc email gốc. Gateway cũng quyết định toolset nào được phép dùng trên từng platform. Ví dụ, một CLI session có thể bật terminal/file tool, còn một bot Telegram nên hạn chế lệnh hệ thống hoặc thao tác file nhạy cảm.

Tool gateway là lớp kết nối với dịch vụ bên ngoài như web search, browser automation, image generation hoặc text-to-speech. Nói ngắn gọn, messaging gateway đưa yêu cầu vào Hermes và trả kết quả ra ngoài; tool gateway giúp Hermes gọi các năng lực bên ngoài trong lúc xử lý.

Có hai kiểu gateway quan trọng:

| Gateway | Vai trò | Ví dụ |
| --- | --- | --- |
| **Messaging Gateway** | Giao tiếp qua nền tảng nhắn tin | Telegram, Discord, Slack, WhatsApp, Email |
| **Tool Gateway** | Dùng dịch vụ bên ngoài | Web search, browser, image generation, text-to-speech |

```mermaid
flowchart TD
    accTitle: Hermes Gateway Flow
    accDescr: Flow showing how a message from an external platform enters Hermes, optionally uses outside services, and returns to the original platform

    message([👤 Tin nhắn từ Telegram hoặc Email]) --> messaging_gateway[🌐 Messaging Gateway nhận tin]
    messaging_gateway --> hermes[🤖 Hermes xử lý bằng LLM, memory, skills, tools]
    hermes --> outside{Cần dịch vụ bên ngoài?}
    outside -->|Có| tool_gateway[🔌 Tool Gateway]
    tool_gateway --> external_service[Web search, browser, image, TTS]
    external_service --> result[📥 Kết quả về Hermes]
    outside -->|Không| result
    result --> response[Hermes tạo câu trả lời]
    response --> send_back[📤 Gateway gửi lại đúng nền tảng ban đầu]

    classDef decision fill:#fef9c3,stroke:#ca8a04,stroke-width:2px,color:#713f12
    classDef process fill:#dbeafe,stroke:#2563eb,stroke-width:2px,color:#1e3a5f
    classDef success fill:#dcfce7,stroke:#16a34a,stroke-width:2px,color:#14532d

    class outside decision
    class messaging_gateway,hermes,tool_gateway,external_service,result,response process
    class send_back success
```

### Ví dụ qua Telegram

Người dùng nhắn:

```text
Kiểm tra giúp tôi hôm nay có tin AI gì mới.
```

Luồng xử lý:

1. Gateway nhận tin nhắn từ Telegram
2. Hermes dùng web tool để tìm tin
3. Hermes tóm tắt kết quả
4. Gateway gửi lại câu trả lời vào Telegram

---

## 👤 Profile: tách agent theo mục đích

Profile cho phép tạo nhiều môi trường Hermes khác nhau trên cùng một máy. Mỗi profile có cấu hình, memory, skills, session, cron jobs và trạng thái gateway riêng.

### Cơ chế hoạt động

Profile tách các không gian làm việc để memory, skills, session và cron job không bị trộn giữa các mục đích khác nhau. Một profile `coder` có thể ưu tiên repo, test và quy tắc code; profile `research` ưu tiên nguồn học thuật; profile `personal` ưu tiên lịch cá nhân và cách giao tiếp.

Khi dựng prompt, Hermes còn đọc context files để hiểu project hiện tại. Các file thường gặp gồm:

```text
.hermes.md / HERMES.md
AGENTS.md
CLAUDE.md
.cursorrules
.cursor/rules/*.mdc
SOUL.md
```

`SOUL.md` là identity/personality toàn cục của Hermes instance và thường được load riêng. Context theo project có thể có priority như `.hermes.md` → `AGENTS.md` → `CLAUDE.md` → `.cursorrules`, để tránh nhét nhiều bộ luật mâu thuẫn vào cùng prompt. Hermes cũng có thể dùng progressive subdirectory discovery: khi agent đi vào thư mục con như `backend/` hoặc `frontend/`, nó mới phát hiện context file riêng trong thư mục đó. Cách này giúp prompt không phình ngay từ đầu và vẫn giữ được hướng dẫn chuyên biệt theo vùng code.

```mermaid
flowchart TD
    accTitle: Hermes Profile Separation
    accDescr: Flow showing how separate Hermes profiles load separate configuration, memory, and skills to avoid mixing contexts across coding, research, and personal work

    choose([👤 Người dùng chọn profile]) --> profile_type{Chọn profile}
    profile_type -->|coder| coder_config[Load cấu hình chuyên code]
    profile_type -->|research| research_config[Load cấu hình nghiên cứu]
    profile_type -->|personal| personal_config[Load cấu hình cá nhân]
    coder_config --> coder_memory[Memory và skills riêng của coder]
    research_config --> research_memory[Memory và skills riêng của research]
    personal_config --> personal_memory[Memory và skills riêng của personal]
    coder_memory --> coder_agent[Hermes làm việc như agent lập trình]
    research_memory --> research_agent[Hermes làm việc như agent nghiên cứu]
    personal_memory --> personal_agent[Hermes làm việc như trợ lý cá nhân]
    coder_agent --> isolated[✅ Không trộn lẫn dữ liệu]
    research_agent --> isolated
    personal_agent --> isolated

    classDef decision fill:#fef9c3,stroke:#ca8a04,stroke-width:2px,color:#713f12
    classDef process fill:#dbeafe,stroke:#2563eb,stroke-width:2px,color:#1e3a5f
    classDef success fill:#dcfce7,stroke:#16a34a,stroke-width:2px,color:#14532d

    class profile_type decision
    class coder_config,research_config,personal_config,coder_memory,research_memory,personal_memory,coder_agent,research_agent,personal_agent process
    class isolated success
```

### Ví dụ profile

| Profile | Nên nhớ gì |
| --- | --- |
| `coder` | Dự án dùng FastAPI, test bằng `pytest`, không push code nếu chưa hỏi |
| `research` | Ưu tiên nguồn học thuật, tóm tắt theo cấu trúc, so sánh nhiều tài liệu |
| `personal` | Người dùng thích câu trả lời ngắn, học tiếng Anh mỗi ngày, nhắc lịch buổi tối |

---

## 🔧 Ví dụ thực tế: dùng Hermes code một MVP nhỏ

Ví dụ trong dữ liệu PDF là một app quản lý chi tiêu cá nhân cho sinh viên Việt Nam. Repo ban đầu có thể như sau:

```text
personal-finance-app/
  backend/
  frontend/
  README.md
```

Người dùng nên thêm `AGENTS.md` để Hermes hiểu luật làm việc trong repo:

```text
personal-finance-app/
  backend/
  frontend/
  README.md
  AGENTS.md
```

### Ví dụ `AGENTS.md`

```markdown
# Project Rules for Hermes

## Goal

Build a personal finance app for Vietnamese students.

## Tech Stack

- Backend: FastAPI
- Database: SQLite
- Frontend: React + Vite
- Styling: simple CSS, no heavy UI library for now

## Coding Rules

- Do not rewrite the whole project unless necessary.
- Always inspect existing files before editing.
- Prefer small commits/changes.
- After coding, run tests or at least run the app/build command.
- Explain what changed in simple Vietnamese.

## Safety

- Do not delete files without asking.
- Do not push to GitHub automatically.
- Do not deploy automatically.
```

### Chạy Hermes trong repo

```bash
cd personal-finance-app
hermes chat --toolsets terminal,skills,web
```

Prompt giao việc:

```text
Hãy tạo MVP app quản lý chi tiêu cá nhân.
Yêu cầu:
- Backend FastAPI
- SQLite
- API thêm/sửa/xóa giao dịch
- Frontend React hiển thị danh sách giao dịch
- Có tổng thu, tổng chi, số dư
- Chạy được local
- Sau khi code xong hãy chạy test/build để kiểm tra
```

### Giải thích quy trình tạo MVP nhỏ

Với yêu cầu tạo MVP, Hermes không nên nhảy thẳng vào viết code tất cả file một lúc. Nó xử lý theo một workflow giống một lập trình viên có kỷ luật:

| Giai đoạn | Hermes làm gì | Kết quả mong muốn |
| --- | --- | --- |
| **1. Hiểu luật và phạm vi** | Load `SOUL.md`, đọc `AGENTS.md`, kiểm tra repo hiện có, xác định stack FastAPI + SQLite + React/Vite và các giới hạn như không deploy, không push tự động | Agent biết cần xây app nhỏ, chạy local, không phá cấu trúc repo |
| **2. Lập kế hoạch MVP** | Chia yêu cầu thành backend, database, API, frontend, state UI và kiểm tra chạy được; tạo TODO ngắn để tránh bỏ sót luồng thêm/sửa/xóa giao dịch | Có checklist rõ ràng thay vì code theo cảm tính |
| **3. Xây backend trước** | Tạo hoặc sửa `main.py`, `database.py`, `models.py`, `schemas.py`, `crud.py`, router giao dịch; đảm bảo API có các endpoint CRUD và tính tổng thu/chi/số dư nếu backend đảm nhiệm | Backend có cấu trúc đủ nhỏ nhưng rõ vai trò từng file |
| **4. Xây frontend sau** | Tạo form nhập giao dịch, danh sách giao dịch, nút sửa/xóa, phần tổng hợp thu chi; kết nối API backend và xử lý trạng thái loading/error cơ bản | Người dùng có giao diện thao tác được, không chỉ có API |
| **5. Kiểm tra bằng tool thật** | Chạy install, test, import check, backend server hoặc frontend build; đọc lỗi nếu có và sửa cho đến khi app chạy local | MVP được xác nhận bằng terminal/build thay vì chỉ nói "đã xong" |
| **6. Tóm tắt và học lại** | Tóm tắt file đã tạo/sửa, cách chạy app, lỗi đã sửa; nếu có quy trình lặp lại, lưu vào `MEMORY.md` hoặc draft `SKILL.md` | Lần sau tạo app tương tự nhanh hơn và ít hỏi lại hơn |

Ví dụ, nếu repo chưa có backend, Hermes có thể bắt đầu bằng một FastAPI app tối thiểu, SQLite local và router `/transactions`. Nếu frontend đã có Vite sẵn, nó không scaffold lại toàn bộ mà đọc cấu trúc hiện có rồi thêm component cần thiết. Đây là điểm quan trọng: Hermes vừa tạo sản phẩm, vừa kiểm soát phạm vi thay đổi.

### Luồng Hermes code dự án

```mermaid
flowchart TD
    accTitle: Hermes Coding Workflow
    accDescr: Flow showing large workflow stages and the smaller execution steps inside each stage

    task([👤 Người dùng giao nhiệm vụ tạo MVP])

    subgraph context["1. Chuẩn bị ngữ cảnh"]
        direction TB
        c1[Load SOUL.md]
        c2[Đọc AGENTS.md]
        c3[Tra compact memory]
        c4[Xác định stack, lệnh test, vùng cấm]
        c1 --> c2 --> c3 --> c4
    end

    subgraph repo["2. Hiểu repo và lập kế hoạch"]
        direction TB
        r1[Quét file tree và README]
        r2[Kiểm tra backend/frontend hiện có]
        r3[Chia TODO: API, DB, UI, test]
        r1 --> r2 --> r3
    end

    subgraph build["3. Triển khai MVP"]
        direction TB
        b1[Đọc file liên quan trước khi sửa]
        b2[Tạo backend: FastAPI, SQLite, CRUD]
        b3[Tạo frontend: form, list, summary]
        b4[Kết nối frontend với API]
        b1 --> b2 --> b4
        b1 --> b3 --> b4
    end

    subgraph verify["4. Kiểm tra bằng môi trường thật"]
        direction TB
        v1[Chạy install nếu cần]
        v2[Chạy test, build hoặc run app]
        v3{Có lỗi?}
        v1 --> v2 --> v3
    end

    subgraph debug["5. Debug nếu thất bại"]
        direction TB
        d1[Đọc error log]
        d2[Khoanh vùng file và dòng lỗi]
        d3[Phân tích nguyên nhân]
        d4[Sửa đúng file liên quan]
        d1 --> d2 --> d3 --> d4
    end

    subgraph learn["6. Tổng kết và học lại"]
        direction TB
        l1[Tóm tắt file đã tạo/sửa]
        l2[Ghi cách chạy local]
        l3{Bài học tái sử dụng được?}
        l4[Lưu MEMORY.md hoặc draft SKILL.md]
        l5([✅ Kết thúc])
        l1 --> l2 --> l3
        l3 -->|Có| l4
        l4 --> l5
        l3 -->|Không| l5
    end

    task -->|bắt đầu phiên| c1
    c4 -->|đủ ngữ cảnh| r1
    r3 -->|có checklist| b1
    b4 -->|sau mỗi cụm thay đổi| v1
    v3 -->|Có lỗi| d1
    d4 -->|kiểm chứng lại| v2
    v3 -->|Không lỗi| l1

    classDef decision fill:#fef9c3,stroke:#ca8a04,stroke-width:2px,color:#713f12
    classDef process fill:#dbeafe,stroke:#2563eb,stroke-width:2px,color:#1e3a5f
    classDef success fill:#dcfce7,stroke:#16a34a,stroke-width:2px,color:#14532d
    classDef stage fill:#f8fafc,stroke:#64748b,stroke-width:2px,color:#0f172a

    class v3,l3 decision
    class c1,c2,c3,c4,r1,r2,r3,b1,b2,b3,b4,v1,v2,d1,d2,d3,d4,l1,l2,l4 process
    class task,l5 success
```

### Cách đọc các ô và thanh nối trong luồng

Mỗi khung lớn là một giai đoạn của workflow. Các ô nhỏ bên trong khung là quá trình thực thi cụ thể mà Hermes làm trong giai đoạn đó. Ví dụ, `Chuẩn bị ngữ cảnh` không chỉ là đọc một file, mà gồm load `SOUL.md`, đọc `AGENTS.md`, tra compact memory, rồi rút ra stack, lệnh test và vùng cấm.

Thanh nối giữa các khung cho biết đầu ra của giai đoạn trước trở thành đầu vào của giai đoạn sau. Nhánh `Có lỗi` đưa Hermes sang cụm debug, sau đó quay lại bước kiểm tra để xác nhận. Nhánh `Không lỗi` mới cho phép đi sang tổng kết và học lại.

### Giải thích quy trình gỡ lỗi trong ví dụ

Điểm khác biệt của Hermes so với chatbot thông thường nằm ở chỗ nó không chỉ nói "có thể lỗi do X". Khi người dùng giao việc như "Hãy kiểm tra xem vì sao một test case của dự án đang bị lỗi", Hermes chạy như một tác nhân bền bỉ qua ba giai đoạn:

| Giai đoạn | Hermes làm gì | Ý nghĩa |
| --- | --- | --- |
| **1. Chuẩn bị ngữ cảnh trước vòng lặp** | Load `SOUL.md` để biết nguyên tắc làm việc cẩn thận; đọc `AGENTS.md` để biết luật repo; xem compact memory để nhớ dự án dùng `pnpm` hay `npm`, lệnh test là gì, lỗi cũ từng gặp ra sao | Agent không cần hỏi lại các thông tin dự án đã biết và bắt đầu với đúng bối cảnh |
| **2. Tương tác thực tế trong vòng lặp** | Dùng terminal chạy test, đọc error log, mở đúng file liên quan, sửa code, chạy lại test/build, rồi lặp cho đến khi hết lỗi hoặc gặp rào cản cần hỏi người dùng | Agent kiểm chứng bằng phản hồi thật từ môi trường thay vì đoán lý thuyết |
| **3. Tự học sau nhiệm vụ** | Nếu phát hiện bài học có giá trị như Docker container bị treo, thiếu biến môi trường riêng của dự án, hoặc lệnh test cần flag đặc biệt, Hermes lưu vào `MEMORY.md` hoặc tạo draft `SKILL.md` | Lần sau gặp lỗi tương tự, agent có thể đề xuất hướng xử lý từ kinh nghiệm cũ thay vì mò lại từ đầu |

Ví dụ, nếu test fail vì thiếu biến môi trường `DATABASE_URL`, một chatbot thường chỉ gợi ý chung chung rằng "hãy kiểm tra env". Hermes có thể đọc log, mở file config, phát hiện biến thiếu, thêm `.env.example` hoặc hướng dẫn chạy đúng lệnh, chạy lại test để xác nhận, rồi ghi nhớ rằng repo này cần `DATABASE_URL` trước khi test backend. Đây là learning loop thực tế: chuẩn bị ngữ cảnh, hành động trong môi trường thật, kiểm chứng kết quả, rồi lưu bài học có chọn lọc.

### Kết quả dự án có thể tạo ra

```text
personal-finance-app/
  backend/
    main.py
    database.py
    models.py
    schemas.py
    crud.py
    requirements.txt
    routers/
      __init__.py
      transactions.py
  frontend/
    package.json
    src/
      App.jsx
      api.js
      components/
        TransactionForm.jsx
        TransactionList.jsx
        SummaryCards.jsx
  AGENTS.md
  README.md
```

### Ví dụ backend chính

```python
# backend/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from database import Base, engine
from routers import transactions

Base.metadata.create_all(bind=engine)

app = FastAPI(title="Personal Finance API")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(transactions.router)
```

### Ví dụ frontend chính

```jsx
// frontend/src/App.jsx
import { useEffect, useState } from "react";
import { getTransactions, createTransaction } from "./api";
import TransactionForm from "./components/TransactionForm";
import TransactionList from "./components/TransactionList";
import SummaryCards from "./components/SummaryCards";

export default function App() {
  const [transactions, setTransactions] = useState([]);

  async function loadData() {
    const data = await getTransactions();
    setTransactions(data);
  }

  useEffect(() => {
    loadData();
  }, []);

  async function handleAdd(formData) {
    await createTransaction(formData);
    await loadData();
  }

  return (
    <main>
      <h1>Personal Finance App</h1>
      <SummaryCards transactions={transactions} />
      <TransactionForm onSubmit={handleAdd} />
      <TransactionList transactions={transactions} />
    </main>
  );
}
```

### Lệnh kiểm tra

```bash
cd backend
pip install -r requirements.txt
uvicorn main:app --reload

cd ../frontend
npm install
npm run build
```

Nếu build hoặc server lỗi, Hermes đọc error log, tìm file liên quan, sửa lại và chạy kiểm tra lần nữa.

---

## 🔍 Vòng sửa lỗi

Cơ chế sửa lỗi của Hermes dựa trên vòng lặp: chạy thử -> đọc lỗi -> tìm nguyên nhân -> sửa code -> chạy lại.

```mermaid
flowchart TD
    accTitle: Hermes Debugging Loop
    accDescr: Flow showing how Hermes uses real terminal feedback and project files to diagnose, patch, and verify code fixes

    fix_request([👤 Người dùng yêu cầu sửa lỗi]) --> inspect_repo[Đọc cấu trúc repo]
    inspect_repo --> run_app[Chạy app, test hoặc build]
    run_app --> error_log[Terminal trả error log]
    error_log --> analyze_error[Hermes phân tích lỗi]
    analyze_error --> read_files[Đọc file liên quan]
    read_files --> patch_code[Sửa code]
    patch_code --> rerun[Chạy lại test hoặc build]
    rerun --> still_error{Còn lỗi?}
    still_error -->|Có| error_log
    still_error -->|Không| report[✅ Báo cáo đã sửa gì]

    classDef decision fill:#fef9c3,stroke:#ca8a04,stroke-width:2px,color:#713f12
    classDef process fill:#dbeafe,stroke:#2563eb,stroke-width:2px,color:#1e3a5f
    classDef success fill:#dcfce7,stroke:#16a34a,stroke-width:2px,color:#14532d

    class still_error decision
    class inspect_repo,run_app,error_log,analyze_error,read_files,patch_code,rerun process
    class report success
```

### Bốn nguồn thông tin Hermes dùng khi sửa lỗi

| Nguồn | Tác dụng |
| --- | --- |
| **Error log** | Cho biết lỗi cụ thể xảy ra ở đâu |
| **Codebase** | Cho biết cấu trúc thật của dự án |
| **Project rules** | Lấy từ `AGENTS.md`, `CLAUDE.md` hoặc `.cursorrules` |
| **Tool feedback** | Kết quả từ terminal, test, build hoặc linter |

### Các loại lỗi có thể xử lý

| Loại lỗi | Ví dụ | Cách Hermes xử lý |
| --- | --- | --- |
| Runtime | `TypeError: 'NoneType' object is not subscriptable` | Tìm dòng lỗi, kiểm tra biến `None`, thêm xử lý điều kiện |
| Test fail | `Expected status code 201, got 422` | Đọc test, đọc endpoint, sửa schema hoặc response code |
| Build/type/lint | `Property 'amount' does not exist on type 'Transaction'` | Đọc interface/type, sửa type hoặc component, chạy lại build |

---

## 🎓 Learning loop: skill hình thành sau nhiều lần code

Sau nhiều nhiệm vụ tương tự, Hermes có thể biến kinh nghiệm thành skill. Ví dụ:

```text
~/.hermes/skills/fastapi-react-mvp/SKILL.md
```

Nội dung skill có thể như sau:

```markdown
# FastAPI React MVP Skill

Use this skill when creating a small full-stack MVP with FastAPI, SQLite, and React.

## Process

1. Inspect repo structure first.
2. Create backend API with FastAPI.
3. Use SQLite for local persistence.
4. Add CORS for local frontend.
5. Create React components:
   - Form
   - List
   - Summary
6. Run backend import check.
7. Run frontend build.
8. Fix errors before final response.

## Do Not

- Do not add authentication unless requested.
- Do not deploy automatically.
- Do not rewrite existing files without checking.
```

Lần sau, khi người dùng nói "Tạo một app CRUD nhỏ bằng FastAPI + React giống lần trước", Hermes có thể tải skill liên quan, làm theo quy trình đã lưu và hoàn thành nhanh hơn.

### Emergent skills và bước kiểm duyệt

Điểm quan trọng của learning loop là Hermes không chỉ ghi nhớ kết quả, mà còn có thể tự sinh tài liệu hướng dẫn sau khi giải xong một bài toán khó. Những kỹ năng này thường được viết thành `SKILL.md` theo kiểu open standard như `agentskills.io`: có phần mô tả khi nào dùng skill, quy trình thao tác, ràng buộc, ví dụ và các điều không nên làm.

Tuy nhiên, skill do AI tự sinh không nên được coi ngay là tri thức chính thức. Ở giai đoạn đầu, chúng nên được xem là `draft`: bản nháp cần con người đọc lại, sửa lại và approve trước khi cho phép agent tự động áp dụng trong các dự án production hoặc môi trường nhạy cảm.

Lý do là agent có thể học lại cả kinh nghiệm đúng lẫn kinh nghiệm sai. Nếu một phiên trước xử lý được lỗi bằng cách vá tạm, bỏ qua test, hoặc áp dụng một giả định chỉ đúng trong bối cảnh hẹp, việc lưu nguyên quy trình đó thành skill có thể khiến Hermes "học vẹt" và lặp lại sai lầm ở dự án khác. Vì vậy, learning loop nên có lớp human-in-the-loop:

| Giai đoạn | Vai trò của Hermes | Vai trò của con người |
| --- | --- | --- |
| Giải bài toán khó | Tìm cách xử lý, chạy kiểm tra, rút kinh nghiệm | Quan sát kết quả và yêu cầu giải thích nếu cần |
| Tạo skill | Viết draft `SKILL.md` từ quy trình đã dùng | Kiểm tra giả định, phạm vi áp dụng và rủi ro |
| Duyệt skill | Đề xuất lưu vào thư mục skills | Approve, sửa, hoặc loại bỏ bản nháp |
| Tái sử dụng | Chỉ tải skill đã được duyệt khi nhiệm vụ phù hợp | Theo dõi các lần áp dụng để cải thiện tiếp |

Nói ngắn gọn: khả năng tự sinh skill giúp Hermes học nhanh hơn, nhưng bước kiểm duyệt giúp hệ thống không biến lỗi cũ thành "kinh nghiệm" chính thức.

---

## 🔐 Lưu ý an toàn khi dùng Hermes

Hermes có thể sửa file, chạy terminal, dùng tool và tự động hóa công việc. Vì vậy, cần đặt ranh giới rõ trước khi giao việc.

| Rủi ro | Cách giảm thiểu |
| --- | --- |
| Agent sửa quá nhiều file | Ghi rõ "prefer small changes" trong `AGENTS.md` |
| Agent xóa file ngoài ý muốn | Ghi rõ "do not delete files without asking" |
| Agent push/deploy tự động | Cấm push/deploy nếu chưa được xác nhận |
| Agent chạy lệnh nguy hiểm | Dùng approval, sandbox, hoặc giới hạn toolset |
| Memory bị lẫn giữa mục đích khác nhau | Tách bằng profile `coder`, `research`, `personal` |

Một `AGENTS.md` tốt nên nói rõ mục tiêu, tech stack, quy tắc code, lệnh kiểm tra và vùng cấm. Đây là cách biến Hermes thành cộng sự có kỷ luật thay vì một agent quá tự do.

---

## 📚 Nguồn dữ liệu

| Tệp PDF | Nội dung chính |
| --- | --- |
| [`1/Giới thiệu Hermes Agent.pdf`](./1/Giới%20thiệu%20Hermes%20Agent.pdf) | Khái niệm AI Agent, Hermes Agent, so sánh với AI Agent nói chung |
| [`1/Cơ chế hoạt động của Hermes Agent.pdf`](./1/Cơ%20chế%20hoạt%20động%20của%20Hermes%20Agent.pdf) | Memory, Skills, Tools, Cron, Gateway, Profile |
| [`1/ví dụ Hermes Agent.pdf`](./1/ví%20dụ%20Hermes%20Agent.pdf) | Ví dụ dùng Hermes code app quản lý chi tiêu cá nhân |

---

## 📌 Kết luận

Hermes Agent có thể hiểu như một runtime cho trợ lý AI lâu dài. Nó không chỉ trả lời câu hỏi mà còn đọc repo, làm theo luật dự án, sửa file, chạy kiểm tra, đọc lỗi, sửa tiếp, ghi nhớ thông tin và biến workflow lặp lại thành skill.

Nói ngắn gọn:

- Chatbot thường: hỏi gì trả lời đó
- AI Agent nói chung: có thể lập kế hoạch và dùng công cụ
- Hermes Agent: có memory, skills, tools, cron, gateway, profile và learning loop để làm việc lâu dài
