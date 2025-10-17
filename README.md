 Activities app (backend, web, mobile)

Short, exact. Follow steps in order. Commands assume project root contains backend/, frontend/, and mobile/ folders. Adjust paths if your layout differs.

Overview

Backend: NestJS GraphQL + Mongoose. Serves HTTP /graphql and WebSocket /graphql (graphql-ws).

Frontend (web): Vite + React + Apollo Client. Uses GraphQL queries, mutations, and subscriptions.

Mobile: Expo (React Native) + Apollo Client. Uses same GraphQL API.

Local Mongo: usually runs in Docker.

Prerequisites

Node.js (v18+ recommended) and npm.

Docker & Docker Desktop (for Mongo).

Expo CLI (optional globally): npm i -g expo-cli or use npx expo.

Recommended ports: backend 3000, frontend 5173 (Vite), Expo web 19006.

Backend (NestJS)
1. Configure

Copy .env.example → .env (if present).

Ensure src/app.module.ts uses your Mongo URI. Default in code:

MongooseModule.forRoot(process.env.MONGO_URI || 'mongodb://127.0.0.1:27017/nest-activities')

2. If using Docker Mongo

Start a Mongo container if you don’t have one:

docker run -d --name mongo -p 27017:27017 -v mongo-data:/data/db mongo:7


Confirm:

docker ps
# expect mongo with port 27017 exposed

3. Ensure CORS and WS host binding

Make sure src/main.ts contains:

app.enableCors({ origin: true });
// or origin: 'http://localhost:5173' while in dev
httpServer.listen(3000, '0.0.0.0', () => { ... });


Reason: allow requests from frontend and physical devices.

4. Install & run
cd backend
npm install
npm run start:dev


Server logs should show Server listening on http://0.0.0.0:3000/graphql (or similar).

5. Clear DB (if needed)

Delete all activities quickly:

# drop database (nuclear)
docker exec -it <mongo_container_name> mongosh --eval "db.getSiblingDB('nest-activities').dropDatabase()"
# or delete a collection
docker exec -it <mongo_container_name> mongosh --eval "db.getSiblingDB('nest-activities').activities.deleteMany({})"

Web Frontend (Vite + React)
1. Configure environment

Create .env at frontend root:

VITE_GRAPHQL_HTTP=http://localhost:3000/graphql
VITE_GRAPHQL_WS=ws://localhost:3000/graphql


If backend runs on a different host/port adjust accordingly.

2. Install & run
cd frontend
npm install
npm run dev


Open: http://localhost:5173

3. Common issues

If subscriptions fail, check browser Network -> ws://.../graphql handshake.

If CORS errors occur, enable CORS in backend or set origin to your frontend dev URL.

If Apollo exports are undefined, clear Vite cache:

rm -rf node_modules/.vite
npm run dev -- --force

Mobile (Expo React Native)
1. Host selection (physical device)

Set host in src/graphql/client.js (or wherever client is configured) to your machine IP on the LAN (example 10.198.162.24).

// for a physical Android device
const HOST = Platform.OS === 'android' ? '10.198.162.24' : (Platform.OS === 'web' ? window.location.hostname : 'localhost');
const HTTP_URL = `http://${HOST}:3000/graphql`;
const WS_URL = (Platform.OS === 'web' ? `${window.location.protocol === 'https:' ? 'wss' : 'ws'}://${HOST}:3000/graphql` : `ws://${HOST}:3000/graphql`);


Find your PC IP: Windows ipconfig → IPv4 under Wi-Fi.

2. Install & run
cd mobile
npm install
npx expo start
# scan QR with Expo Go (make sure device and PC are on same Wi-Fi)

3. Run on web (for testing)
npx expo start --web
# Visit URL printed by Expo (usually http://localhost:19006)


When running on web, client uses window.location.hostname to set backend host automatically.

4. Troubleshooting

If app cannot reach server from device: ensure backend listening on 0.0.0.0 and firewall not blocking port 3000.

If subscriptions fail on device: check backend logs and ensure graphql-ws server created with path: '/graphql'.

If Hermes/WebSocket differences occur on old SDKs, use polling fallback: useQuery(GET_ACTIVITIES, { pollInterval: 5000 }).

GraphQL operations (examples)

Create

mutation Create($input: CreateActivityInput!) {
  createActivity(input: $input) {
    id title message category createdAt expiresAt
  }
}


Fetch

query {
  getActivities { id title message category createdAt expiresAt }
}


Subscribe

subscription {
  activityCreated { id title message category createdAt }
}

Recommended dev workflow

Start Mongo (Docker).

Start backend: cd backend && npm run start:dev.

Start frontend: cd frontend && npm run dev.

Start mobile (if needed): cd mobile && npx expo start.


Test create in frontend or mobile. Confirm subscription updates appear on other clients.
