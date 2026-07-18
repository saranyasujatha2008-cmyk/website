# website
Built using: Stitch, Claude, and ChatGPT Purpose: An AI-powered career roadmap generator that creates personalized learning paths based on a user's current skill level.
import express from "express";
import path from "path";
import { GoogleGenAI, Type } from "@google/genai";
import dotenv from "dotenv";

dotenv.config();

const app = express();
const PORT = 3000;

app.use(express.json());

// Initialize Gemini client safely
let ai: GoogleGenAI | null = null;
const geminiApiKey = process.env.GEMINI_API_KEY;

if (geminiApiKey) {
  try {
    ai = new GoogleGenAI({
      apiKey: geminiApiKey,
      httpOptions: {
        headers: {
          'User-Agent': 'aistudio-build',
        }
      }
    });
    console.log("Gemini AI Client initialized successfully.");
  } catch (error) {
    console.error("Failed to initialize Gemini AI client:", error);
  }
} else {
  console.log("No GEMINI_API_KEY env variable found. Using high-quality mock fallbacks.");
}

// -------------------------------------------------------------------------
// Gemini Retry and Fallback Utilities (to resiliently handle 503 / 429 errors)
// -------------------------------------------------------------------------

async function callWithRetry<T>(fn: () => Promise<T>, maxRetries = 3, initialDelay = 500): Promise<T> {
  let delay = initialDelay;
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (err: any) {
      console.warn(`Attempt ${attempt}/${maxRetries} failed with error:`, err?.message || err);
      
      const isTransient = err?.status === "UNAVAILABLE" || 
                          err?.message?.includes("503") || 
                          err?.message?.includes("429") || 
                          err?.message?.includes("RESOURCE_EXHAUSTED") ||
                          err?.message?.includes("high demand") ||
                          err?.message?.includes("overloaded");

      if (attempt < maxRetries && isTransient) {
        console.log(`Transient error detected. Retrying in ${delay}ms...`);
        await new Promise((resolve) => setTimeout(resolve, delay));
        delay *= 2;
      } else {
        throw err;
      }
    }
  }
  throw new Error("Retry logic failed to resolve.");
}

async function generateContentWithRetryAndFallback(params: {
  contents: any;
  config?: any;
  primaryModel?: string;
}) {
  if (!ai) {
    throw new Error("Gemini AI Client is not initialized.");
  }

  const primaryModel = params.primaryModel || "gemini-3.5-flash";
  const modelsToTry = [
    primaryModel,
    "gemini-flash-latest",
    "gemini-3.1-flash-lite"
  ];

  let lastError: any = null;

  for (const model of modelsToTry) {
    try {
      const response = await callWithRetry(async () => {
        console.log(`Requesting Gemini using model: ${model}`);
        return await ai!.models.generateContent({
          model: model,
          contents: params.contents,
          config: params.config,
        });
      }, 3, 500);
      
      console.log(`Successfully generated content using model: ${model}`);
      return response;
    } catch (err: any) {
      lastError = err;
      console.warn(`Model ${model} failed completely after retries. Swapping to next fallback...`);
    }
  }

  throw lastError || new Error("All fallback models failed.");
}

// -------------------------------------------------------------------------
// Helper functions & Fallback Mock Generators (Extremely rich & high-fidelity)
// -------------------------------------------------------------------------

function getMockRoadmap(targetRole: string, skills: string[], weeklyHours: number): any {
  const roleName = targetRole || "Full Stack Developer";
  const userSkillsStr = skills.length > 0 ? skills.join(", ") : "HTML, CSS, JavaScript";
  
  return [
    {
      id: "phase-1",
      phaseNumber: 1,
      phaseName: "Core Foundations & Architectural Gaps",
      phaseDuration: "3 Months",
      items: [
        {
          id: "item-1-1",
          title: `Mastering Advanced ${roleName === "AI Engineer" ? "Python & PyTorch" : "TypeScript & State Design"}`,
          durationHours: 45,
          status: "Completed",
          skillsCovered: roleName === "AI Engineer" ? ["Python", "PyTorch", "TensorFlow"] : ["TypeScript", "Generics", "State Management", "XState"],
          subskills: [
            { name: roleName === "AI Engineer" ? "Deep Learning Basics" : "Advanced Utility Types", completed: true },
            { name: roleName === "AI Engineer" ? "Neural Network Architecture" : "Finite State Machines & XState", completed: true },
            { name: "Asynchronous Design Patterns", completed: true }
          ],
          weeks: "Weeks 1-4",
          description: "Establish a pristine mental model of advanced asynchronous programming, compile-time assertions, and modular design."
        },
        {
          id: "item-1-2",
          title: roleName === "AI Engineer" ? "NLP & Transformer Architectures" : "High-Performance Core Web Vitals",
          durationHours: 60,
          status: "Active",
          skillsCovered: roleName === "AI Engineer" ? ["LLMs", "HuggingFace", "Prompt Eng"] : ["Core Web Vitals", "Lighthouse", "Performance API", "CLS/LCP"],
          subskills: [
            { name: roleName === "AI Engineer" ? "Attention Mechanisms" : "Lazy Loading & Tree Shaking", completed: true },
            { name: roleName === "AI Engineer" ? "Fine-Tuning Models" : "Cumulative Layout Shift Optimization", completed: false },
            { name: roleName === "AI Engineer" ? "RAG Architectures" : "Server-Side Rendering Diagnostics", completed: false }
          ],
          weeks: "Weeks 5-10",
          description: `Direct focus on the critical areas matching your target path to ${roleName}. Bridging skills from current foundational level.`
        }
      ]
    },
    {
      id: "phase-2",
      phaseNumber: 2,
      phaseName: "Architecture, Systems & Production Scale",
      phaseDuration: "6 Months",
      items: [
        {
          id: "item-2-1",
          title: roleName === "AI Engineer" ? "Model Pipelines & MLOps Infrastructure" : "Micro-Frontends & Module Federation",
          durationHours: 80,
          status: "Next Up",
          skillsCovered: roleName === "AI Engineer" ? ["MLOps", "Docker", "Kubeflow"] : ["Webpack 5", "Vite", "Mono-repos", "Turborepo"],
          subskills: [
            { name: roleName === "AI Engineer" ? "Dockerizing Models" : "Module Federation Config", completed: false },
            { name: roleName === "AI Engineer" ? "CI/CD for Machine Learning" : "Independent Bundle Deployments", completed: false },
            { name: "Containerization & Scaling", completed: false }
          ],
          weeks: "Weeks 11-18",
          description: "Transitioning isolated code solutions into distributed micro-architectures that scale horizontally across clusters."
        },
        {
          id: "item-2-2",
          title: roleName === "AI Engineer" ? "Vector Databases & Semantics" : "System Design & Distributed Data",
          durationHours: 120,
          status: "Next Up",
          skillsCovered: roleName === "AI Engineer" ? ["Pinecone", "ChromaDB", "Semantic Search"] : ["GraphQL", "Caching Strategies", "Redis", "Database Sharding"],
          subskills: [
            { name: roleName === "AI Engineer" ? "Vector Embeddings Generation" : "API Gateway Orchestration", completed: false },
            { name: "Redis In-Memory Caching", completed: false },
            { name: "Schema Federation & Subgraphs", completed: false }
          ],
          weeks: "Weeks 19-24",
          description: "Understanding network overhead, storage redundancy, and querying optimized datastores."
        }
      ]
    },
    {
      id: "phase-3",
      phaseNumber: 3,
      phaseName: "System Leadership & Mastery Focus",
      phaseDuration: "12 Months",
      items: [
        {
          id: "item-3-1",
          title: "Technical Strategy & Engineering Leadership",
          durationHours: 150,
          status: "Next Up",
          skillsCovered: ["Engineering Leadership", "Technical Debt Mapping", "RFC Documentation", "Mentorship"],
          subskills: [
            { name: "Writing Technical RFCs", completed: false },
            { name: "System Threat Modeling & Audits", completed: false },
            { name: "Cross-functional Alignments", completed: false }
          ],
          weeks: "Weeks 25-36",
          description: "Operating as an active architect of technical strategy. Drafting blueprints, justifying investments, and leveling up peers."
        }
      ]
    }
  ];
}

// -------------------------------------------------------------------------
// API ROUTES
// -------------------------------------------------------------------------

// 1. Generate Career Roadmap
app.post("/api/generate-roadmap", async (req, res) => {
  const { targetRole, deadline, weeklyHours, skills, education, learningStyle } = req.body;
  
  if (!ai) {
    console.log("Generating mock roadmap...");
    const roadmap = getMockRoadmap(targetRole, skills, weeklyHours);
    return res.json({ roadmap });
  }

  try {
    const prompt = `You are a legendary AI Technical Career Path Architect. 
Your objective is to generate an absolute state-of-the-art, high-fidelity career roadmap for an aspiring candidate targeting the role of: "${targetRole}".

Candidate details:
- Target Role: ${targetRole}
- Career Timeline: Target deadline is ${deadline}
- Weekly Commitment: ${weeklyHours} hours
- Current Technical Foundation: ${skills.join(", ")}
- Education Background: ${education}
- Preferred learning style: ${learningStyle}

Design a personalized, highly optimized career roadmap divided into 3 distinct chronological phases (Phase 1: Foundation & Urgent Gaps, Phase 2: Architecture & Scale, Phase 3: Leadership & Mastery).
Each phase MUST contain 1 to 3 learning items. Each learning item MUST have a set of interactive subskills.
Keep the description highly technical, specific, and motivating (No generic advice. Use exact technology words like Vite, Redis, PyTorch, Docker, Core Web Vitals, etc.).

Return the response strictly as a JSON object of type:
{
  "roadmap": [
    {
      "id": "string",
      "phaseNumber": 1,
      "phaseName": "string",
      "phaseDuration": "string",
      "items": [
        {
          "id": "string",
          "title": "string",
          "durationHours": 45,
          "status": "Completed" | "Active" | "Next Up",
          "skillsCovered": ["string"],
          "subskills": [
            { "name": "string", "completed": false }
          ],
          "weeks": "string",
          "description": "string"
        }
      ]
    }
  ]
}`;

    const response = await generateContentWithRetryAndFallback({
      contents: prompt,
      config: {
        responseMimeType: "application/json",
        responseSchema: {
          type: Type.OBJECT,
          required: ["roadmap"],
          properties: {
            roadmap: {
              type: Type.ARRAY,
              description: "The chronologically ordered phases of the roadmap.",
              items: {
                type: Type.OBJECT,
                required: ["id", "phaseNumber", "phaseName", "phaseDuration", "items"],
                properties: {
                  id: { type: Type.STRING },
                  phaseNumber: { type: Type.INTEGER },
                  phaseName: { type: Type.STRING },
                  phaseDuration: { type: Type.STRING },
                  items: {
                    type: Type.ARRAY,
                    items: {
                      type: Type.OBJECT,
                      required: ["id", "title", "durationHours", "status", "skillsCovered", "subskills", "weeks", "description"],
                      properties: {
                        id: { type: Type.STRING },
                        title: { type: Type.STRING },
                        durationHours: { type: Type.INTEGER },
                        status: { type: Type.STRING, enum: ["Completed", "Active", "Next Up"] },
                        skillsCovered: { type: Type.ARRAY, items: { type: Type.STRING } },
                        subskills: {
                          type: Type.ARRAY,
                          items: {
                            type: Type.OBJECT,
                            required: ["name", "completed"],
                            properties: {
                              name: { type: Type.STRING },
                              completed: { type: Type.BOOLEAN }
                            }
                          }
                        },
                        weeks: { type: Type.STRING },
                        description: { type: Type.STRING }
                      }
                    }
                  }
                }
              }
            }
          }
        }
      }
    });

    if (response.text) {
      const parsed = JSON.parse(response.text.trim());
      return res.json(parsed);
    } else {
      throw new Error("No response text from Gemini");
    }
  } catch (error) {
    console.error("Gemini Roadmap Error:", error);
    // Return high-quality mock roadmap on failure
    const roadmap = getMockRoadmap(targetRole, skills, weeklyHours);
    return res.json({ roadmap, note: "Fallback used due to API rate-limit or error." });
  }
});

// 2. AI Project Companion Guide Builder
app.post("/api/generate-project-guide", async (req, res) => {
  const { projectTitle, difficulty, skills, userStack } = req.body;

  if (!ai) {
    return res.json({
      guide: {
        architecture: "Distributed server-client setup with Redis pub/sub channels and a PostgreSQL datastore.",
        databaseSchema: `CREATE TABLE users (\n  id UUID PRIMARY KEY,\n  email VARCHAR(255) UNIQUE,\n  created_at TIMESTAMP DEFAULT NOW()\n);\n\nCREATE TABLE items (\n  id UUID PRIMARY KEY,\n  user_id UUID REFERENCES users(id),\n  payload JSONB,\n  updated_at TIMESTAMP\n);`,
        steps: [
          "Phase 1: Set up the folder scaffolding and initialize standard typescript compiler configurations.",
          "Phase 2: Configure your Docker files and establish a clean local Redis instance for pub/sub messaging.",
          "Phase 3: Write the core service modules in your preferred backend framework using strict interface contracts.",
          "Phase 4: Integrate standard telemetry logs and deploy with continuous integration checks."
        ]
      }
    });
  }

  try {
    const prompt = `You are a Senior Staff Engineer. Generate a custom, incredibly detailed development implementation companion for a student portfolio project: "${projectTitle}" (${difficulty} difficulty).
Current user skills/stack to align with: ${userStack ? userStack.join(", ") : "TypeScript, Node.js"}.
Skills targeted by this project: ${skills.join(", ")}.

Provide:
1. Recommended System Architecture Overview (Keep it professional, mentioning microservices, APIs, caches where applicable).
2. Direct SQL or schema definition (e.g. Postgres DDL or JSON schemas) that fits this project.
3. Logical sequential steps for implementation (3-5 detailed phases).

Format your output strictly as a JSON object matching:
{
  "guide": {
    "architecture": "A brief overview string describing design components.",
    "databaseSchema": "SQL / Schema code block as a string.",
    "steps": ["Step 1 description", "Step 2 description", ...]
  }
}`;

    const response = await generateContentWithRetryAndFallback({
      contents: prompt,
      config: {
        responseMimeType: "application/json",
        responseSchema: {
          type: Type.OBJECT,
          required: ["guide"],
          properties: {
            guide: {
              type: Type.OBJECT,
              required: ["architecture", "databaseSchema", "steps"],
              properties: {
                architecture: { type: Type.STRING },
                databaseSchema: { type: Type.STRING },
                steps: { type: Type.ARRAY, items: { type: Type.STRING } }
              }
            }
          }
        }
      }
    });

    if (response.text) {
      const parsed = JSON.parse(response.text.trim());
      return res.json(parsed);
    } else {
      throw new Error("Empty text in Gemini response");
    }
  } catch (error) {
    console.error("Gemini Project Companion Error:", error);
