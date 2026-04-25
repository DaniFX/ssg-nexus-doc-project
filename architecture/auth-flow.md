# 🔐 Nexus Auth Flow

1. **Client**: Effettua login tramite Firebase SDK.
2. **Gateway**: Intercetta la richiesta con header `Authorization: Bearer <JWT>`.
3. **Gateway**: Verifica il JWT con `firebase-admin` SDK.
4. **Enrichment**: 
   - Estrae `uid` -> `X-Nexus-User-ID`.
   - Estrae custom claims `role` -> `X-Nexus-Role`.
5. **Dispatch**: Inoltra la richiesta al microservizio Cloud Run tramite rete VPC interna (Ingress: Internal + Cloud Load Balancing).