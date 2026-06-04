# Post-Mortem: Incidente de Error 500 en Endpoint /owners

**Fecha:** 2026-06-03
**Duracion:** 5 minutos
**Severidad:** SEV-2
**Autores:** Equipo proyecto-devops

## Resumen ejecutivo
Un cambio de codigo introdujo un bug en el endpoint /owners que retornaba HTTP 500. El error fue desplegado a produccion por el pipeline CD. Fue detectado mediante el dashboard de Grafana (aumento de error rate) y resuelto con rollback automatico via kubectl.

## Timeline (UTC)
| Hora | Evento |
|------|--------|
| 21:00 | Commit con bug es pusheado a main |
| 21:03 | Pipeline CI pasa (bug no es de compilacion) |
| 21:04 | Pipeline CD despliega a staging |
| 21:05 | Error rate sube al 100% en dashboard Grafana |
| 21:06 | Equipo detecta el incidente |
| 21:07 | Ejecutar: kubectl rollout undo deployment/petclinic |
| 21:08 | Rollback completado. Error rate vuelve a 0% |
| 21:10 | Incidente cerrado. Servicio restaurado |

## Root Cause Analysis

### Causa raiz
Se introdujo una excepcion no manejada (NullPointerException) en el metodo findOwner() del OwnerController. El codigo no validaba el caso de un ID inexistente.

### 5 Whys
1. **Por que fallo el endpoint?** Porque el codigo lanzaba NullPointerException
2. **Por que no se detecto en CI?** Porque los unit tests no cubrian el caso de ID inexistente
3. **Por que no habia test para ese caso?** Porque no se definieron edge cases en el plan de testing
4. **Por que no se validaba el input?** Porque no habia validacion defensiva en el controller
5. **Por que no habia validacion?** Porque no existia un checklist de code review con reglas de validacion

## Impacto
- **Usuarios afectados:** Todos los que accedian a /owners durante 5 minutos
- **Requests fallidos:** ~50 requests con HTTP 500
- **SLO impactado:** Disponibilidad cayo al 0% durante el incidente
- **Error budget consumido:** ~0.01% del presupuesto mensual

## Action Items
| Accion | Responsable | Prioridad | Fecha limite |
|--------|------------|-----------|-------------|
| Agregar test para ID inexistente | Dev team | Alta | 1 semana |
| Implementar validacion defensiva en controllers | Dev team | Alta | 1 semana |
| Agregar smoke test automatico post-deploy | DevOps | Media | 2 semanas |
| Crear checklist de code review | Tech Lead | Media | 2 semanas |
| Configurar alerta automatica en Grafana para error rate > 1% | SRE | Alta | 1 semana |

## Lecciones aprendidas
- El pipeline CI puede pasar incluso con bugs logicos si los tests no cubren edge cases
- El rollback rapido de Kubernetes es critico para mantener el MTTR bajo
- La observabilidad (Grafana) fue clave para detectar el incidente en menos de 2 minutos
- Se necesitan smoke tests automaticos como parte del pipeline CD
