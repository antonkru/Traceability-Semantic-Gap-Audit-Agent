# Human in the loop approval feature

In a high-stakes **IEC 62304** environment, human approval is rarely a binary "Yes/No." Treating this step as a **State Machine** rather than a checkbox allows the AI to react intelligently to your feedback, ensuring that safety-critical logic isn't just "rejected," but actively "remediated."

Here are four suggested levels of human interaction for your agentic workflow:

---

## **1\. The Human Approval "Menu"**

Instead of a simple boolean, your **Coordinator Agent** should listen for specific "Action Directives."

| Option | Meaning | Regulatory Impact |
| :---- | :---- | :---- |
| **"Approved"** | You accept the code and the AI's reasoning. | **Final Sign-off:** Logged as your professional verification. |
| **"Request Clarification"** | You aren't sure about the AI's logic. | **Query State:** The agent must provide deeper "Chain of Thought" or cross-reference another document. |
| **"Conditional Approval"** | Approved *if* a specific minor change is made. | **Pending State:** Trigger a re-audit of a specific sub-module only. |
| **"Rejected \- Safety Risk"** | High-level failure (e.g., race condition in Class C module). | **Block State:** Code is flagged; automated prevention of merge. |

---

## **2\. Actions When a Human Disapproves**

When you hit "Reject," you shouldn't have to start over. The workflow should enter a **Remediation Loop**.

### **A. The "Alternative Logic" Branch**

The **Coordinator** instructs the **Code Inspector** to generate three alternative implementations that specifically address your objection (e.g., "Implement this using a Mutex instead of a lock to avoid the deadlock risk I identified").

### **B. The "Risk Register" Escalation**

If you reject code because of an unforeseen side effect, the **Risk Agent** should automatically update your **ISO 14971 Risk Register**. It documents: *"Human Expert identified a new failure mode during AI-assisted review. Mitigation required."*

### **C. The "Deep-Dive" Audit**

The workflow triggers a more expensive, high-reasoning model (e.g., moving from a fast 3.5-level model to a flagship 4.0-level model) to perform a formal mathematical verification or an exhaustive scan of the codebase to see if the rejected logic exists elsewhere.

---

## **3\. Designing the "Human-in-the-Loop" State Machine**

This diagram illustrates how the workflow transitions based on your feedback:

1. **Drafting State:** Specialist agents produce code and verification.  
2. **Review State:** Human evaluates the "Compliance Package."  
3. **Path A (Approve):** Extractor Agent archives the log $\\rightarrow$ Merge code.  
4. **Path B (Query):** Specialist Agent expands on the reasoning $\\rightarrow$ Back to Review.  
5. **Path C (Reject):** Risk Agent flags the commit $\\rightarrow$ Re-generation loop $\\rightarrow$ Back to Review.

---

## **4\. The "Auditor's View" of Rejections**

From a compliance perspective, **recorded rejections are actually good.**

If an auditor sees a log where 100% of AI suggestions were approved, they will suspect "Rubber Stamping." If they see logs where you rejected the AI's first attempt because it missed a specific safety constraint, it proves that your **Human Oversight (EU AI Act Art. 14\)** is functional and rigorous.

