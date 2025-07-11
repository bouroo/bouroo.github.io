---
title: "Let's Build a Voting System with Go and Svelte"
subtitle: ""
date: 2025-01-01T00:00:00+07:00
lastmod: 2025-01-01T00:00:00+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: ""
license: ""
images: []

tags: []
categories: []

featuredImage: "featured-image.jpg"
featuredImagePreview: "featured-image.jpg"

lightgallery: true
---

Lately, I've been seeing a lot of local government election campaigns, so I took the opportunity to pick up a web technology that seems to be overlooked, Server-Sent Events (SSE), to create a simple online voting system that supports real-time score display using **Go** for the backend and **Svelte** for the frontend.

<!--more-->

## Server-Sent Events

Normally, a web page has to send a request to the server to get new data. That is, the web page requests data from the server. But with Server-Sent Events (SSE), the server can send data to the client in real-time via the HTTP protocol (PUSH) without the client having to request data every time. This is different from WebSockets, which open a two-way (full-duplex) connection. SSE sends data from the server to the client in one direction only (one-way).

{{< mermaid >}}
graph TD;
    A[Client] -->|HTTP Request| B[Server]
    B -->|HTTP Header| F[Content-Type: text/event-stream]
    F -->|HTTP Response| A
    B -->|Sends Events| C[Event Stream]
    C -->|Updates| A
    A -->|Handles Events| D[JavaScript Event Listener]
    D -->|Processes Data| E[Update UI]
{{< /mermaid >}}

### How SSE works
1. Establishing a connection: When a client wants to receive data from the server, it sends an HTTP GET request to the server.
2. Sending data: The server responds by sending data in the form of an event stream using the `Content-Type: text/event-stream` header.
3. Updating data: The server continuously sends data (events) to the client when new data becomes available.
4. Handling received data: The client uses JavaScript to wait for events (event listener) and process the received data to update the UI or perform other tasks.

## Explaining the functions in Go

### 1. `NewVoteManager()`

This function is a constructor for creating a new `VoteManager`, which will have default values for candidates and the voting channel.

```go
func NewVoteManager() *VoteManager {
    vm := &VoteManager{
        candidates: map[string]*Candidate{
            "Candidate A": {Name: "Candidate A", Votes: 0},
            "Candidate B": {Name: "Candidate B", Votes: 0},
        },
        voteChannel: make(chan string, runtime.NumCPU()*2),
        clients:     make(map[chan string]struct{}),
        cliRequests: make(chan cliRequest),
    }
    go vm.manageClients()
    return vm
}
```

**How it works**: When the server starts, it creates a `VoteManager` with two candidates, "Candidate A" and "Candidate B", with an initial score of 0.

---

### 2. `Start()`

This function starts the `VoteManager` by opening a `goroutine` to wait for votes from the `voteChannel`.

```go
func (vm *VoteManager) Start(ctx context.Context) {
    vm.wg.Add(1)
    go func() {
        defer vm.wg.Done()
        for {
            select {
            case candidateName, ok := <-vm.voteChannel:
                if !ok {
                    return
                }
                vm.processVote(candidateName)
            case <-ctx.Done():
                return
            }
        }
    }()
}
```

**How it works**: When this function is called, the server starts receiving votes from users. If a user votes for "Candidate A", the server will send this candidate's name to the `processVote` function.

---

### 3. `processVote()`

This function checks if the voted candidate exists in the system and increments the score for the corresponding candidate.

```go
func (vm *VoteManager) processVote(candidateName string) {
    if candidate, exists := vm.candidates[candidateName]; exists {
        candidate.Votes++
        vm.notifyClients(candidate)
    } else {
        log.Printf("Received vote for unknown candidate: %s", candidateName)
    }
}
```

**How it works**: If a vote is cast for "Candidate A", the score will increase from 0 to 1, and all users will be notified of the new score. The score will continue to increase with each vote for the candidate.

---

### 4. `notifyClients()`

This function sends the updated score data to all connected users.

```go
func (vm *VoteManager) notifyClients(candidate *Candidate) {
    message, err := json.Marshal(candidate)
    if err != nil {
        log.Printf("Failed to marshal candidate: %v", err)
        return
    }

    for clientChan := range vm.clients {
        select {
        case clientChan <- string(message):
        default:
            log.Println("Skipping sending to a slow client")
        }
    }
}
```

**How it works**: When the score of "Candidate A" increases, this function sends the new score data to all users through the connected channel, indicating the current score of "Candidate A".

---

### 5. `voteHandler()`

This function is responsible for handling vote requests from users. It receives the candidate's name from the URL parameters.

```go
func (vm *VoteManager) voteHandler(w http.ResponseWriter, r *http.Request) {
    candidateName := r.URL.Query().Get("candidate")
    if candidateName == "" {
        http.Error(w, "Candidate name is required", http.StatusBadRequest)
        return
    }
    select {
    case vm.voteChannel <- candidateName:
        w.WriteHeader(http.StatusAccepted)
    default:
        http.Error(w, "Server is busy, try again later", http.StatusServiceUnavailable)
    }
}
```

**How it works**: If a user sends a `GET` request to `/vote?candidate=Candidate A`, this function will send the name "Candidate A" to the `voteChannel` and return a 202 (Accepted) status. If the server cannot accept the vote, it will return a 503 (Service Unavailable) status.

---

### 6. `resultsHandler()`

This function returns the current scores of all candidates in JSON format.

```go
func (vm *VoteManager) resultsHandler(w http.ResponseWriter, r *http.Request) {
    candidateList := make([]*Candidate, 0, len(vm.candidates))
    for _, candidate := range vm.candidates {
        c := &Candidate{
            Name:  candidate.Name,
            Votes: candidate.Votes,
        }
        candidateList = append(candidateList, c)
    }
    if err := json.NewEncoder(w).Encode(candidateList); err != nil {
        http.Error(w, "Failed to encode results", http.StatusInternalServerError)
    }
}
```

**How it works**: When a user accesses `/results`, the server will send a JSON object of all candidates and their scores, for example:

```json
[
    {"name": "Candidate A", "votes": 1},
    {"name": "Candidate B", "votes": 0}
]
```

---

### 7. `sseHandler()`

This function is used to handle Server-Sent Events (SSE) connections, allowing users to receive real-time data.

```go
func (vm *VoteManager) sseHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("Connection", "keep-alive")

    flusher, ok := w.(http.Flusher)
    if !ok {
        http.Error(w, "Streaming unsupported!", http.StatusInternalServerError)
        return
    }

    clientChan := make(chan string, runtime.NumCPU()*2)
    vm.AddClient(clientChan)
    defer vm.RemoveClient(clientChan)

    initialData, err := json.Marshal(vm.candidates)
    if err == nil {
        w.Write([]byte("data: " + string(initialData) + "\n\n"))
        flusher.Flush()
    }

    notify := r.Context().Done()
    pingTicker := time.NewTicker(1 * time.Minute)
    defer pingTicker.Stop()

    for {
        select {
        case msg, ok := <-clientChan:
            if !ok {
                return
            }
            if _, err := w.Write([]byte("data: " + msg + "\n\n")); err != nil {
                log.Println("Error writing to client:", err)
                return
            }
            flusher.Flush()

        case <-notify:
            return

        case <-pingTicker.C:
            if _, err := w.Write([]byte(":\n\n")); err != nil {
                log.Println("Error during ping:", err)
                return
            }
            flusher.Flush()
        }
    }
}
```

**How it works**: When a client establishes a connection to `/events`, the server sends all candidate data in JSON format to the client and also sends updated score data when a new vote is cast.

---

## Explaining the functions in Svelte

### 1. `fetchResults()`

This function is used to fetch the candidates' score data from the server.

```javascript
async function fetchResults() {
    loading = true;
    try {
        const response = await axios.get("http://localhost:8080/results");
        candidates = response.data.sort((a, b) => a.name.toLowerCase().localeCompare(b.name.toLowerCase()));
    } catch (error) {
        errorMessage = "Error fetching results. Please try again later.";
        console.error("Error fetching results:", error);
    } finally {
        loading = false;
    }
}
```

**How it works**: When the page loads, this function is called to fetch the candidates' score data and store it in the `candidates` variable for display.

---

### 2. `vote()`

This function is called when the user clicks the vote button.

```javascript
async function vote(candidate) {
    voting = true;
    errorMessage = "";
    try {
        await axios.get(`http://localhost:8080/vote?candidate=${candidate}`);
        voted = true;
        setTimeout(() => {
            voted = false;
        }, 5000);
    } catch (error) {
        errorMessage = "Error voting. Please try again.";
        console.error("Error voting:", error);
    } finally {
        voting = false;
    }
}
```

**How it works**: If the user clicks to vote for "Candidate A", this function sends a request to the server and sets the `voted` variable to `true` to prevent repeated voting within a 5-second period, simulating a new user casting a vote.

---

### 3. `setupSSE()`

This function is used to set up an SSE connection to receive real-time score data.

```javascript
function setupSSE() {
    const eventSource = new EventSource("http://localhost:8080/events");

    eventSource.onmessage = function (event) {
        const updatedCandidate = JSON.parse(event.data);
        const index = candidates.findIndex(c => c.name === updatedCandidate.name);
        if (index !== -1) {
            candidates[index].votes = updatedCandidate.votes;
        }
    };

    eventSource.onerror = function (err) {
        console.error("EventSource failed:", err);
        eventSource.close();
    };
}
```

**How it works**: When the score changes, this function receives the updated score data from the server and automatically updates the `candidates` in the UI.

---

### Example of how it works

![example](img/example.gif "example")

## Conclusion

We can apply SSE to simple real-time display applications, such as updating scores by pushing new data from the server, which eliminates the need to refresh data from the client and does not require opening a WebSocket. You can see the full example code at [bouroo/sse-voting-app](https://github.com/bouroo/sse-voting-app)
