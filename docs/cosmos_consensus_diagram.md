```mermaid
flowchart TD
    Start([New Height/Round]) --> PS[Proposer Selection]
    PS --> TC[Transaction Collection]
    TC --> PP[ABCI: PrepareProposal]
    PP --> BP[Broadcast Proposal]
    
    BP --> RCV[Validators Receive Proposal]
    RCV --> VAL{Validate Block Structure}
    VAL -->|Valid| PROC[ABCI: ProcessProposal]
    VAL -->|Invalid| NILPV[Prevote NIL]
    
    PROC --> ACCEPT{App Accepts?}
    ACCEPT -->|Yes| EV[ABCI: ExtendVote]
    ACCEPT -->|No| NILPV
    
    EV --> PV[Broadcast Prevote + Extension]
    NILPV --> PV
    
    PV --> COLLECT[Collect All Prevotes]
    COLLECT --> VER[ABCI: VerifyVoteExtension]
    VER --> POLKA{>2/3 Prevotes for Block?}
    
    POLKA -->|Yes - Polka!| PC[Precommit Block]
    POLKA -->|No Polka| TIMEOUT1{Timeout?}
    TIMEOUT1 -->|Yes| NEXTROUND[Next Round]
    TIMEOUT1 -->|No| COLLECT
    
    PC --> BPC[Broadcast Precommit]
    BPC --> COLLECTPC[Collect Precommits]
    COLLECTPC --> COMMIT{>2/3 Precommits?}
    
    COMMIT -->|Yes| FB[ABCI: FinalizeBlock]
    COMMIT -->|No| TIMEOUT2{Timeout?}
    TIMEOUT2 -->|Yes| NEXTROUND
    TIMEOUT2 -->|No| COLLECTPC
    
    FB --> CMT[ABCI: Commit]
    CMT --> SAVE[Save Block to Disk]
    SAVE --> NEXTH[Next Height]
    NEXTH --> Start
    
    NEXTROUND --> PS
    
    %% Styling
    classDef abciCall fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    classDef consensus fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    classDef decision fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef broadcast fill:#e8f5e8,stroke:#1b5e20,stroke-width:2px
    
    class PP,PROC,EV,VER,FB,CMT abciCall
    class PS,TC,COLLECT,COLLECTPC,SAVE consensus
    class VAL,ACCEPT,POLKA,COMMIT,TIMEOUT1,TIMEOUT2 decision
    class BP,PV,BPC broadcast
    ```
