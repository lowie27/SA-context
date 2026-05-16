# Patient Monitoring System (PMS) - Architecture Extension

This repository contains the software architecture documentation and models for the PMS Project (Part 2). The project focuses on evolving an initial architecture to meet high-priority performance and availability requirements.

## Project Structure

- `/lectures` & `/book`: Theoretical foundation based on Software Architecture in Practice.
- `/project/`
    - `SA_project_appendixA_requirements.pdf`: is a core project document that defines the Functional Requirements for the Patient Monitoring System (PMS). It serves as the primary reference for the system's "Problem Space"—the specific services the system must provide and the behavior it must exhibit.
    - `SA_project_appendixB_APIs.pdf`: is a technical project document that specifies the healthAPI, the primary interface for the Patient Monitoring System (PMS) to interoperate with the external Hospital Information System (HIS).
    - `SA_project_description.pdf`: serves as the Domain Description for the Patient Monitoring Service (PMS) project. It is the foundational document that outlines the high-level context, business objectives, and the overarching "Problem Space" for the system you are designing.
- `/project/part1` : part1 is finished so no need to worry about this!
- `/project/part2-current`: Active workspace. Contains current .vpp files and extension buildsheets.
- `/project/part2-archive`: Old/Deprecated. Not used for current development.
- `/project/visual-paradigm`: Tooling support and the mandatory SAPlugin.
    - `SAPluginManual-10.0.0.pdf`: details a custom extension for Visual Paradigm (VP) designed to support the Software Architecture course by automating consistency checks and the generation of LaTeX-based reports.
- `/project/part2-current/`
    - `SA_project_part2_inital_architecture_including_rationale.pdf`: provides a detailed description of the starting point for your architecture extension, documenting the decisions, tactics, and trade-offs made to satisfy two initial Quality Attribute Scenarios (QAS)
        - `Scenario P2`: Risk Estimation (Performance)
        - `Scenario M2`: New Clinical Models (Modifiability)
    - `SA_project_part2_instructions.pdf`: outlines the requirements and methodology for the quality-driven architectural extension of the initial Patient Monitoring Service (PMS). The primary objective is to evolve the starting design to fully support four specific Quality Attribute Scenarios (QASs) while explicitly reasoning about the resulting architectural trade-offs.

## Current Focus: Part 2 Extension

We are currently replacing the OtherFunctionality placeholder with a concrete design to support:

- `Scenario Av2 (Availability)`: Detecting communication failures between the PatientGateway and backend within 5 seconds using Ping/Echo tactics.
    - teammates part, do not edit
- `Scenario P1 (Performance)`: Ensuring physician requests for high-risk data are processed in 2 seconds without impacting raw sensor data ingress.
    - my part this is what we will be working on
