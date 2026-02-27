## Usage of AI

I used AI heavily throughout this project, primarily through the Codex CLI. This repo is documentation-heavy rather than code-heavy, so AI was most useful for drafting, restructuring, and iterating on design material quickly. I used it as a tool to accelerate exploration and writing, not as a substitute for deciding what the system should actually be.

My usage was roughly as follows:

1. I started by working through the assignment and deciding what the core shape of the credit verification system should be.
2. I used Codex to help draft the design document, dataflow document, and schema structure.
3. I repeatedly revised the design to reduce jargon, make the writing more narrative, and keep the system focused on the actual verification problem instead of adding unnecessary framework language.
4. I used AI to explore how to explain the pipeline, the review flow, and the handling of mixed evidence types such as PDFs, GeoJSON, and source-system records.
5. I also used it to draft and then trim the verification schema so that it stayed readable for reviewers instead of becoming unnecessarily elaborate.

Areas where I was most manually involved were:

1. Simplifying the writing whenever the generated output became too abstract, too technical, or too generic.
2. Pushing the design toward a more realistic explanation of messy evidence and human review, rather than a clean but artificial system description.
3. Deciding what belonged in the README versus what should stay in a separate dataflow document.
4. Trimming schema complexity and removing fields or structures that felt like artificial bloat.
5. Reworking examples and narrative sections so the document sounded more human and less like generated boilerplate.

This repo changed a lot through manual iteration. The higher-level architecture, tooling choices, and overall stack direction were not invented by AI. I explicitly provided that direction myself, including the general system shape, the type of services involved, and example architectural patterns to follow. AI was most useful on the verification-specific material, especially schema iteration, case structure, and drafting how the evidence and review flow should be described. I was more directly responsible for the infrastructure framing, tooling choices, and concrete implementation direction, including explicit stack decisions and examples such as the AWS-oriented deployment patterns. The final structure reflects repeated human direction on tone, scope, and simplification, and the main judgment calls were about what to remove, what to clarify, and what level of complexity was actually appropriate for this assignment.
