# Agronomy Test Module

## Purpose
Sales managers were taking sales agents' tests via Google Forms which was hard to manage. This module replaces that — create questions, create tests from those questions, launch tests, agents answer them, system auto-marks objective questions, subjective ones need manual evaluation.

## Flow
1. **Create Questions** — 3 types: `mcq`, `fill in the blank`, `subjective`
2. **Create Test** — attach questions, assign to agents (by specific IDs, agent_category, or state), set live_on date
3. **Agents Take Test** — submit answer sheets; system auto-marks MCQ & fill-in-the-blank (has predefined answer), subjective stays unchecked
4. **Manual Marking** — admins manually mark subjective (unchecked) answers via `/marking` endpoint
5. **Launch Result** — once all sheets are checked, result can be launched with rankings & notifications

## File Structure
```
routes/agronomyTest/
├── route.cjs                              — Route definitions
├── controller/agronomyTest.cjs            — All endpoint handlers
├── validators/
│   ├── index.cjs                          — Validator exports
│   ├── validateCreateQuestions.cjs         — Create question validation
│   ├── validateCreateAnswerSheet.cjs       — Create answer sheet validation
│   ├── validateCreateTest.cjs             — Create test validation
│   ├── validateUpdateTest.cjs             — Update test validation
│   └── validateUpdateQuestions.cjs         — Update question validation
└── utils/
    ├── marksCalculator.cjs                — Sums marks from question IDs
    ├── pendingAgentHelper.cjs             — Resolves target agents from specific_agents/category/state
    ├── checkExamAndAgentCompatibility.cjs — Checks if agent is allowed to give the test
    └── answerSheetCheckNotification.cjs   — Sends notification + socket event after checking/result launch

models/
├── agronomyTestQuestionsModel.cjs         — Question schema
├── agronomyTestModel.cjs                  — Test schema
└── agronomyTestAnswerSheetModel.cjs       — Answer sheet schema

swaggerDocs/agronomyTest.cjs              — Swagger API docs
```

## Schemas

### Question (`agronomy_question`)
- `question_type`: enum `["fill in the blank", "mcq", "subjective"]`
- `question`: string (required)
- `image_urls`: [String]
- `answer`: string (null for subjective)
- `mcq_options`: [String] (only for mcq)
- `difficulty`: enum `["easy", "medium", "hard"]` — maps to marks: easy=1, medium=3, hard=5
- `marks`: enum `[1, 3, 5]` — auto-set based on difficulty
- `topic`, `created_by`: strings

### Test (`agronomy_test`)
- `test_id`: auto-generated `TEST-{sequence}` (uppercase)
- `test_name`, `total_marks`, `agent_category`, `state`
- `specific_agents`: [katyayani_ids]
- `questions`: [ObjectId] ref agronomy_question
- `live_on`: Date — test goes live on this date, can't update/delete after
- `pending_agents`: agents who haven't taken the test yet
- `given_by`: agents who have submitted
- `result_launched`, `result_launched_at`, `result_launched_by`
- `tags`, `month`, `created_by`

### Answer Sheet (`agronomy_answer_sheet`)
- `katyayani_id`, `test_id`
- `answers`: [{ question_id, submitted_answer, marks_obtained, difficulty }]
- `total_marks_obtained`, `out_of`
- `is_checked`: false if any subjective question exists (needs manual marking)
- `evaluated_by`: defaults to "SYSTEM", changes to admin's ID on manual marking

## API Endpoints
| Method | Path | Access | Description |
|--------|------|--------|-------------|
| GET | `/` | All | Get tests (paginated, filterable) |
| GET | `/question` | All | Get questions (paginated, filterable) |
| GET | `/answerSheet` | All (agents see own, admins see all) | Get answer sheets |
| POST | `/` | All | Create test |
| POST | `/question` | All | Create question |
| POST | `/answerSheet` | All | Submit answer sheet |
| POST | `/result` | All | View/launch test results |
| PATCH | `/` | Admin/Agronomist | Update test (before live_on) |
| PATCH | `/question` | Admin/Agronomist | Update question |
| PUT | `/marking` | Admin/Agronomist | Manually mark subjective answers |
| DELETE | `/` | Admin/Agronomist | Delete test or question |

## Key Business Rules
- Test cannot be updated/deleted after `live_on` date passes
- Agent can only submit one answer sheet per test
- Cannot submit answer sheet after result is launched
- Only agents in `pending_agents` can take the test
- Result can only be launched when ALL answer sheets are checked
- Marks: easy=1, medium=3, hard=5 (auto-set from difficulty)
- MCQ & fill-in-the-blank: auto-marked (case-insensitive comparison)
- Subjective: needs manual marking, `is_checked` stays false until marked

## Recent Fixes (Feb 2026)
- Fixed model typos (`deafault` → `default`, `deafult` → `default`)
- Fixed `quesitons` typo in update_test + added missing `await` on calculate_total_marks
- Fixed notification param mismatch (`marks_obtained` → `total_marks_obtained`)
- Fixed case-insensitive comparison inconsistency in answer sheet marking
- Fixed wrong MongoDB query in delete question usage check
- Added question type guards: subjective rejects answer/mcq_options, fill-in-the-blank rejects mcq_options (validators + controller)
- Removed unused `monthsArray` from answer sheet validator
