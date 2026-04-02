# CarkedItOnline

Multiplayer card game web application.

**[CarkedIt.com](https://carkedit.com)** | [Client Repo](https://github.com/bh679/carkedit-online) | [API Repo](https://github.com/bh679/carkedit-api) | [Project Board](https://github.com/users/bh679/projects/10)

---

## Project Structure

This is the **orchestrator repo** containing project-level configuration, Playwright E2E tests, and the Product Engineer workflow.

| Repo | Purpose |
|---|---|
| [CarkedIt](https://github.com/bh679/CarkedIt) | Orchestrator — tests, config, workflow |
| [carkedit-online](https://github.com/bh679/carkedit-online) | Frontend client |
| [carkedit-api](https://github.com/bh679/carkedit-api) | Game server (Node.js + Colyseus) |

## Setup

```bash
npm install
```

## Testing

```bash
npm test              # run Playwright E2E tests
npm run test:report   # view test report
```

## License

ISC
