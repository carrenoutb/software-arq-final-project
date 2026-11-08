# Fase 3 — Implementación de la decisión

*Proyecto de ejemplo · Feature dummy: **Reservas de libros** en `biblioteca-api`*

> **Resultado** de la Fase 3, siguiendo la decisión del ADR-001 (Opción A: tabla + módulo por capas paralelo a `loans`). Todo el código respeta el patrón real del proyecto (Go + Gin + sqlx, capas `handler → service → repository`, JWT con roles). Se muestra archivo por archivo con explicación, como el taller del endpoint `/ping`.

---

## 0. Plan de archivos (nuevos y modificados)

| Archivo | Tipo | Contenido |
|---|---|---|
| `migrations/002_reservations.sql` | **nuevo** | Tabla `reservations` + índices |
| `internal/models/reservation.go` | **nuevo** | Struct `Reservation`, `ReservationRequest`, `ReservationDetail` |
| `internal/repository/reservation_repository.go` | **nuevo** | SQL de reservas |
| `internal/service/reservation_service.go` | **nuevo** | Reglas de negocio (fila, duplicado, avance) |
| `internal/handler/reservation_handler.go` | **nuevo** | Endpoints HTTP |
| `internal/handler/routes.go` | **editar** | Registrar rutas de reservas |
| `internal/service/loan_service.go` | **editar** | `Return()` → avanza la fila |
| `cmd/api/main.go` | **editar** | Wiring (repo → service → handler) |

> ⚠️ **Regla del proyecto:** si hubiera que tocar `books`, `loans` o la migración 001, se haría solo si el ADR lo justifica. Aquí NO se tocan: el préstamo existente funciona igual.

---

## 1. Migración nueva (`migrations/002_reservations.sql`)

```sql
-- Reservas de libros sin copias disponibles (fila de espera)
SET NAMES utf8mb4;

CREATE TABLE IF NOT EXISTS reservations (
  id INT AUTO_INCREMENT PRIMARY KEY,
  user_id INT NOT NULL,
  book_id INT NOT NULL,
  status ENUM('waiting', 'fulfilled', 'canceled') NOT NULL DEFAULT 'waiting',
  reserved_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES users(id),
  FOREIGN KEY (book_id) REFERENCES books(id)
);

-- La fila se atiende por antigüedad: la reserva más vieja 'waiting' es la primera
CREATE INDEX idx_reservations_book_status ON reservations (book_id, status, reserved_at);
```

**¿Qué pasó?** Se creó la tabla con su propio dominio (`status` con 3 estados de reserva — no contamina `loans.status`), FKs hacia `users` y `books`, y un índice que permite consultar "dame la reserva `waiting` más antigua del libro X" sin escanear toda la tabla. Nota: MySQL ejecuta `002` después de `001` automáticamente (mecanismo `docker-entrypoint-initdb.d`), en orden alfabético.

---

## 2. Modelo (`internal/models/reservation.go`)

```go
package models

import "time"

// Reservation representa la espera de un usuario por un libro sin copias disponibles.
type Reservation struct {
	ID         int       `db:"id" json:"id"`
	UserID     int       `db:"user_id" json:"user_id"`
	BookID     int       `db:"book_id" json:"book_id"`
	Status     string    `db:"status" json:"status"`
	ReservedAt time.Time `db:"reserved_at" json:"reserved_at"`
}

// ReservationRequest es el payload para reservar un libro.
type ReservationRequest struct {
	BookID int `json:"book_id" binding:"required" example:"1"`
}

// ReservationDetail combina la reserva con la info del libro (para mostrar al usuario el título).
type ReservationDetail struct {
	Reservation
	BookTitle  string `db:"book_title" json:"book_title"`
	BookAuthor string `db:"book_author" json:"book_author"`
}
```

**¿Qué pasó?** Misma forma que `models/loan.go`: struct de dominio, request con `binding:"required"` (validación HTTP en la capa correcta) y un `Detail` con JOIN al libro para que el frontend no haga doble petición.

---

## 3. Repository (`internal/repository/reservation_repository.go`)

```go
package repository

import (
	"time"

	"github.com/jmoiron/sqlx"

	"biblioteca-api/internal/models"
)

type ReservationRepository struct {
	db *sqlx.DB
}

func NewReservationRepository(db *sqlx.DB) *ReservationRepository {
	return &ReservationRepository{db: db}
}

func (r *ReservationRepository) Create(userID, bookID int) (*models.Reservation, error) {
	query := `INSERT INTO reservations (user_id, book_id, status) VALUES (?, ?, 'waiting')`
	result, err := r.db.Exec(query, userID, bookID)
	if err != nil {
		return nil, err
	}
	id, err := result.LastInsertId()
	if err != nil {
		return nil, err
	}
	return r.FindByID(int(id))
}

func (r *ReservationRepository) FindByID(id int) (*models.Reservation, error) {
	var res models.Reservation
	if err := r.db.Get(&res, `SELECT * FROM reservations WHERE id = ?`, id); err != nil {
		return nil, err
	}
	return &res, nil
}

// HasWaiting indica si el usuario ya tiene una reserva activa (waiting) del mismo libro.
func (r *ReservationRepository) HasWaiting(userID, bookID int) (bool, error) {
	var count int
	query := `SELECT COUNT(*) FROM reservations WHERE user_id = ? AND book_id = ? AND status = 'waiting'`
	if err := r.db.Get(&count, query, userID, bookID); err != nil {
		return false, err
	}
	return count > 0, nil
}

// FindWaitingByUser devuelve las reservas activas de un usuario, con datos del libro.
func (r *ReservationRepository) FindWaitingByUser(userID int) ([]models.ReservationDetail, error) {
	query := `
		SELECT r.*, b.title AS book_title, b.author AS book_author
		FROM reservations r
		JOIN books b ON b.id = r.book_id
		WHERE r.user_id = ? AND r.status = 'waiting'
		ORDER BY r.reserved_at ASC`
	var rows []models.ReservationDetail
	if err := r.db.Select(&rows, query, userID); err != nil {
		return nil, err
	}
	return rows, nil
}

// FindAll devuelve la fila completa por libro (uso administrativo).
func (r *ReservationRepository) FindAll() ([]models.ReservationDetail, error) {
	query := `
		SELECT r.*, b.title AS book_title, b.author AS book_author
		FROM reservations r
		JOIN books b ON b.id = r.book_id
		WHERE r.status = 'waiting'
		ORDER BY r.book_id ASC, r.reserved_at ASC`
	var rows []models.ReservationDetail
	if err := r.db.Select(&rows, query); err != nil {
		return nil, err
	}
	return rows, nil
}

func (r *ReservationRepository) Cancel(id int) error {
	query := `UPDATE reservations SET status = 'canceled' WHERE id = ? AND status = 'waiting'`
	_, err := r.db.Exec(query, id)
	return err
}

// FulfillOldest marca como 'fulfilled' la reserva waiting más antigua de un libro.
// Devuelve true si había alguien esperando (para que el servicio informe al libro).
func (r *ReservationRepository) FulfillOldest(bookID int) (bool, error) {
	query := `
		UPDATE reservations
		SET status = 'fulfilled'
		WHERE id = (
			SELECT id FROM (
				SELECT id FROM reservations
				WHERE book_id = ? AND status = 'waiting'
				ORDER BY reserved_at ASC
				LIMIT 1
			) AS oldest
		)`
	result, err := r.db.Exec(query, bookID)
	if err != nil {
		return false, err
	}
	affected, err := result.RowsAffected()
	if err != nil {
		return false, err
	}
	return affected > 0, nil
}

// Marca con timestamp de atención (opcional, para visibilidad de la fila).
func (r *ReservationRepository) TouchFulfilledAt(id int) error {
	_, err := r.db.Exec(`UPDATE reservations SET fulfilled_at = ? WHERE id = ?`, time.Now(), id)
	return err
}
```

**¿Qué pasó?** El repository solo sabe SQL. Nota dos detalles: `Cancel` tiene `AND status = 'waiting'` (no se cancela una reserva ya atendida — la restricción vive en SQL para que nadie la saltee) y `FulfillOldest` usa un sub-select "copia" (`SELECT id FROM (...)`) porque MySQL no permite `UPDATE … SELECT` sobre la misma tabla directamente — error clásico que se evita con este patrón.

---

## 4. Servicio (`internal/service/reservation_service.go`)

```go
package service

import (
	"errors"

	"biblioteca-api/internal/models"
	"biblioteca-api/internal/repository"
)

var (
	ErrBookHasCopies       = errors.New("el libro aún tiene copias disponibles, no se puede reservar")
	ErrAlreadyWaiting      = errors.New("ya tienes una reserva activa de este libro")
	ErrReservationNotFound = errors.New("reserva no encontrada")
	ErrCannotCancel        = errors.New("solo se pueden cancelar reservas en espera")
)

type ReservationService struct {
	reservationRepo *repository.ReservationRepository
	bookRepo        *repository.BookRepository
}

func NewReservationService(reservationRepo *repository.ReservationRepository, bookRepo *repository.BookRepository) *ReservationService {
	return &ReservationService{reservationRepo: reservationRepo, bookRepo: bookRepo}
}

// Reserve crea la reserva solo si el libro no tiene copias y no hay reserva duplicada.
func (s *ReservationService) Reserve(userID int, req models.ReservationRequest) (*models.Reservation, error) {
	book, err := s.bookRepo.FindByID(req.BookID)
	if err != nil {
		return nil, err
	}
	if book.AvailableCopies > 0 {
		return nil, ErrBookHasCopies
	}

	already, err := s.reservationRepo.HasWaiting(userID, req.BookID)
	if err != nil {
		return nil, err
	}
	if already {
		return nil, ErrAlreadyWaiting
	}

	return s.reservationRepo.Create(userID, req.BookID)
}

func (s *ReservationService) MyReservations(userID int) ([]models.ReservationDetail, error) {
	return s.reservationRepo.FindWaitingByUser(userID)
}

func (s *ReservationService) AllReservations() ([]models.ReservationDetail, error) {
	return s.reservationRepo.FindAll()
}

func (s *ReservationService) Cancel(userID int, reservationID int) error {
	res, err := s.reservationRepo.FindByID(reservationID)
	if err != nil {
		return ErrReservationNotFound
	}
	if res.UserID != userID {
		return errors.New("no puedes cancelar una reserva de otro usuario")
	}
	if res.Status != "waiting" {
		return ErrCannotCancel
	}
	return s.reservationRepo.Cancel(reservationID)
}

// AdvanceQueue atiende la primera reserva waiting del libro (llamado por LoanService.Return).
func (s *ReservationService) AdvanceQueue(bookID int) (bool, error) {
	ok, err := s.reservationRepo.FulfillOldest(bookID)
	if err != nil {
		return false, err
	}
	return ok, nil
}
```

**¿Qué pasó?** Las **reglas de negocio viven aquí**, igual que en `LoanService`: validar copias antes de reservar, rechazar duplicado activo, y el hook `AdvanceQueue` — una operación **semántica** ("avanza la fila") que `loans` invoca sin conocer el esquema de `reservations`.

---

## 5. Handler (`internal/handler/reservation_handler.go`)

```go
package handler

import (
	"net/http"
	"strconv"

	"github.com/gin-gonic/gin"

	"biblioteca-api/internal/models"
	"biblioteca-api/internal/service"
)

type ReservationHandler struct {
	reservationService *service.ReservationService
}

func NewReservationHandler(reservationService *service.ReservationService) *ReservationHandler {
	return &ReservationHandler{reservationService: reservationService}
}

// Reserve godoc
// @Summary      Reservar un libro agotado
// @Tags         reservations
// @Security     BearerAuth
// @Router       /reservations [post]
func (h *ReservationHandler) Reserve(c *gin.Context) {
	userID := c.GetInt("user_id")

	var req models.ReservationRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	reservation, err := h.reservationService.Reserve(userID, req)
	if err != nil {
		c.JSON(http.StatusConflict, gin.H{"error": err.Error()})
		return
	}

	c.JSON(http.StatusCreated, reservation)
}

func (h *ReservationHandler) MyReservations(c *gin.Context) {
	userID := c.GetInt("user_id")
	reservations, err := h.reservationService.MyReservations(userID)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "no se pudieron obtener las reservas"})
		return
	}
	c.JSON(http.StatusOK, reservations)
}

func (h *ReservationHandler) AllReservations(c *gin.Context) {
	reservations, err := h.reservationService.AllReservations()
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "no se pudieron obtener las reservas"})
		return
	}
	c.JSON(http.StatusOK, reservations)
}

func (h *ReservationHandler) Cancel(c *gin.Context) {
	userID := c.GetInt("user_id")
	id, err := strconv.Atoi(c.Param("id"))
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "id inválido"})
		return
	}

	if err := h.reservationService.Cancel(userID, id); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	c.JSON(http.StatusOK, gin.H{"message": "reserva cancelada correctamente"})
}
```

**¿Qué pasó?** El handler **no decide nada**: toma `user_id` del contexto (lo puso el middleware JWT), valida el body, delega en el servicio y traduce errores a códigos HTTP. `Cancel` usa `c.Param("id")` igual que `loans/:id/return`.

---

## 6. Rutas (`internal/handler/routes.go` — edición)

Se agregan dentro de los grupos existentes:

```go
// En el grupo protegido (después de loanHandler)
protected.GET("/reservations/me", reservationHandler.MyReservations)
protected.POST("/reservations", reservationHandler.Reserve)
protected.DELETE("/reservations/:id", reservationHandler.Cancel)

// En el grupo admin
admin.GET("/reservations", reservationHandler.AllReservations)
```

**¿Qué pasó?** Las rutas de reservas reutilizan los **mismos middleware** ya probados (`AuthRequired` y `AdminRequired`): `POST /api/reservations` y `GET /api/reservations/me` requieren token; `GET /api/reservations` (fila completa) solo admin. No se creó infraestructura de seguridad nueva: se compone lo existente.

---

## 7. Hook de integración (`internal/service/loan_service.go` — edición mínima)

Al final de `Return()`, después de `MarkReturned` e `IncrementAvailable`:

```go
// LoanService ahora conoce a ReservationService solo por su operación de fila.
func (s *LoanService) SetReservationService(rs *service.ReservationService) {
	s.reservationService = rs
}

// Dentro de Return(), después de incrementar copias:
if s.reservationService != nil {
	// Se devolvió una copia: avanza la fila de reservas de ese libro.
	s.reservationService.AdvanceQueue(loan.BookID)
}
```

**¿Qué pasó?** El préstamo no sabe qué es una reserva: solo llama a una operación semántica. Ese "puente" se inyecta en `main.go` (wiring), igual que ya se inyectan repositorios y servicios entre sí.

---

## 8. Wiring (`cmd/api/main.go` — edición)

```go
reservationRepo := repository.NewReservationRepository(database)
reservationService := service.NewReservationService(reservationRepo, bookRepo)
reservationHandler := handler.NewReservationHandler(reservationService)

// Bridge: la devolución avanza la fila de reservas.
loanService.SetReservationService(reservationService)

handler.SetupRoutes(router, authHandler, bookHandler, loanHandler, reservationHandler, cfg.JWTSecret)
```

**¿Qué pasó?** Todo se arma en `main` (compositor), igual que el resto: repos → services → handlers → rutas. El `SetupRoutes` ahora recibe `reservationHandler` como parámetro adicional.

---

## ✅ Cierre de la implementación

Con esto quedan cumplidos los requisitos funcionales:

| Requisito | Endpoint | Capas tocadas |
|---|---|---|
| Reservar libro agotado | `POST /api/reservations` | handler → service → repository |
| Mis reservas | `GET /api/reservations/me` | igual |
| Cancelar reserva propia | `DELETE /api/reservations/:id` | igual (valida dueño + estado) |
| Fila completa (admin) | `GET /api/reservations` | igual |
| Avanzar la fila al devolver | (hook en `LoanService.Return`) | loan_service → reservation_service |

**Pendiente:** compilar, levantar y correr la verificación completa (Fase 4), incluyendo confirmar que **los libros y préstamos funcionan igual que antes**.