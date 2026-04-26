# Ehtic-pay-v4เข้าใจครับ คุณต้องการโค้ดเต็มระบบที่พร้อมใช้งานจริงสำหรับการซื้อขายสินค้า/บริการ โดยมีการทำธุรกรรมครบวงจรทั้งฝั่ง TRON และ EVM

ผมจะส่งให้เป็นโครงสร้างโปรเจคแบบสมบูรณ์ พร้อมทุกไฟล์ที่จำเป็น

---

# **Payment System Full Implementation**

## **Project Structure**
```
payment-system/
├── cmd/
│   └── worker/
│       └── main.go
├── internal/
│   ├── tron/
│   │   ├── client.go
│   │   ├── observer.go
│   │   └── models.go
│   ├── evm/
│   │   ├── client.go
│   │   ├── observer.go
│   │   └── cache.go
│   ├── worker/
│   │   ├── reconciler.go
│   │   └── job.go
│   ├── admin/
│   │   ├── handler.go
│   │   └── routes.go
│   ├── notification/
│   │   ├── slack.go
│   │   ├── email.go
│   │   └── alert.go
│   ├── db/
│   │   ├── migrations.go
│   │   └── queries.go
│   └── config/
│       └── config.go
├── frontend/
│   └── admin/
│       ├── index.html
│       └── app.js
├── go.mod
└── go.sum
```

## **1. Config (config.go)**
```go
package config

import (
	"os"
	"strconv"
	"time"
)

type Config struct {
	// Database
	DBHost     string
	DBPort     int
	DBUser     string
	DBPassword string
	DBName     string

	// TRON
	TronAPIKey     string
	TronPrimaryURL string
	TronFallbackURL string

	// EVM
	EvmRPCURL      string
	EvmUSDTAddress string
	EvmChainID     int

	// Worker
	WorkerInterval      time.Duration
	ConfirmationTarget  int
	MaxRetryAttempts    int

	// Notification
	SlackWebhookURL string
	SMTPHost        string
	SMTPPort        int
	SMTPUser        string
	SMTPPassword    string
	AdminEmail      string

	// Admin
	AdminPort int
	JWTSecret string
}

func Load() *Config {
	cfg := &Config{
		DBHost:              getEnv("DB_HOST", "localhost"),
		DBPort:              getEnvInt("DB_PORT", 5432),
		DBUser:              getEnv("DB_USER", "payment"),
		DBPassword:          getEnv("DB_PASSWORD", "password"),
		DBName:              getEnv("DB_NAME", "payment_db"),
		TronAPIKey:          getEnv("TRON_API_KEY", ""),
		TronPrimaryURL:      getEnv("TRON_PRIMARY_URL", "https://api.trongrid.io"),
		TronFallbackURL:     getEnv("TRON_FALLBACK_URL", "https://api.mainnet.trongrid.io"),
		EvmRPCURL:           getEnv("EVM_RPC_URL", "https://mainnet.infura.io/v3/your-project-id"),
		EvmUSDTAddress:      getEnv("EVM_USDT_ADDRESS", "0xdAC17F958D2ee523a2206206994597C13D831ec7"),
		EvmChainID:          getEnvInt("EVM_CHAIN_ID", 1),
		WorkerInterval:      getEnvDuration("WORKER_INTERVAL", "30s"),
		ConfirmationTarget:  getEnvInt("CONFIRMATION_TARGET", 12),
		MaxRetryAttempts:    getEnvInt("MAX_RETRY_ATTEMPTS", 10),
		SlackWebhookURL:     getEnv("SLACK_WEBHOOK_URL", ""),
		SMTPHost:            getEnv("SMTP_HOST", "smtp.gmail.com"),
		SMTPPort:            getEnvInt("SMTP_PORT", 587),
		SMTPUser:            getEnv("SMTP_USER", ""),
		SMTPPassword:        getEnv("SMTP_PASSWORD", ""),
		AdminEmail:          getEnv("ADMIN_EMAIL", "admin@example.com"),
		AdminPort:           getEnvInt("ADMIN_PORT", 8080),
		JWTSecret:           getEnv("JWT_SECRET", "your-secret-key"),
	}
	return cfg
}

func getEnv(key, defaultVal string) string {
	if val := os.Getenv(key); val != "" {
		return val
	}
	return defaultVal
}

func getEnvInt(key string, defaultVal int) int {
	if val := os.Getenv(key); val != "" {
		if i, err := strconv.Atoi(val); err == nil {
			return i
		}
	}
	return defaultVal
}

func getEnvDuration(key, defaultVal string) time.Duration {
	if val := os.Getenv(key); val != "" {
		if d, err := time.ParseDuration(val); err == nil {
			return d
		}
	}
	d, _ := time.ParseDuration(defaultVal)
	return d
}
```

## **2. TRON Client (tron/client.go)**
```go
package tron

import (
	"bytes"
	"context"
	"crypto/sha3"
	"encoding/hex"
	"encoding/json"
	"fmt"
	"math/big"
	"net/http"
	"strings"
	"sync"
	"time"

	"github.com/fbsobreira/gotron-sdk/pkg/address"
	"github.com/shopspring/decimal"
)

const (
	USDTContractBase58 = "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t"
	USDTDecimals       = 6
)

type Client struct {
	APIKey     string
	PrimaryURL string
	FallbackURL string
	HTTP       *http.Client
	mu         sync.RWMutex
	blockCache map[int64]time.Time
}

func New(apiKey, primaryURL, fallbackURL string) *Client {
	return &Client{
		APIKey:     apiKey,
		PrimaryURL: primaryURL,
		FallbackURL: fallbackURL,
		HTTP: &http.Client{
			Timeout: 15 * time.Second,
			Transport: &http.Transport{
				MaxIdleConns:    100,
				MaxConnsPerHost: 100,
				IdleConnTimeout: 90 * time.Second,
			},
		},
		blockCache: make(map[int64]time.Time),
	}
}

type TxObservation struct {
	TxHash        string
	ToAddress     string
	Amount        decimal.Decimal
	Confirmations int64
	Success       bool
	TokenContract string
	BlockNumber   int64
	BlockTime     time.Time
}

// ObserveUSDTTransferTo ดึงข้อมูลธุรกรรมตาม txid และ decode TRC20 Transfer event
func (c *Client) ObserveUSDTTransferTo(ctx context.Context, txHash, toWalletBase58 string) (TxObservation, error) {
	wantTo, err := normalizeBase58(toWalletBase58)
	if err != nil {
		return TxObservation{}, fmt.Errorf("invalid target address: %w", err)
	}

	// ลอง primary ก่อน ถ้าไม่สำเร็จค่อย fallback
	bases := []struct {
		url    string
		useKey bool
	}{
		{c.PrimaryURL, true},
		{c.FallbackURL, false},
	}

	var lastErr error
	for _, b := range bases {
		obs, err := c.observeOnce(ctx, b.url, b.useKey, txHash, wantTo)
		if err == nil {
			return obs, nil
		}
		lastErr = err
	}
	return TxObservation{}, lastErr
}

func (c *Client) observeOnce(ctx context.Context, baseURL string, useKey bool, txHash, wantTo string) (TxObservation, error) {
	// 1. ดึง tx info (receipt + logs + blockNumber)
	txInfo, err := c.getTxInfo(ctx, baseURL, useKey, txHash)
	if err != nil {
		return TxObservation{}, fmt.Errorf("get tx info failed: %w", err)
	}

	if txInfo.ID == "" || txInfo.BlockNumber <= 0 {
		return TxObservation{}, fmt.Errorf("tx not found yet")
	}

	// 2. ดึง latest block
	latest, err := c.getNowBlock(ctx, baseURL, useKey)
	if err != nil {
		return TxObservation{}, fmt.Errorf("get now block failed: %w", err)
	}

	confs := int64(0)
	if txInfo.BlockNumber > 0 {
		confs = latest - txInfo.BlockNumber + 1
		if confs < 0 {
			confs = 0
		}
	}

	success := (txInfo.Receipt.Result == "SUCCESS")

	// 3. Decode logs หา Transfer event ของ USDT
	t0 := strings.ToLower(transferTopic0())

	for _, lg := range txInfo.Log {
		if len(lg.Topics) < 3 {
			continue
		}
		
		// ตรวจสอบ topic0 == Transfer(address,address,uint256)
		if strings.ToLower(lg.Topics[0]) != t0 {
			continue
		}

		// ตรวจสอบ contract address == USDT
		contractB58, err := tronContractHexToBase58(lg.Address)
		if err != nil {
			continue
		}
		if contractB58 != USDTContractBase58 {
			continue
		}

		// ตรวจสอบ to address
		to20, err := last20BytesFromTopic(lg.Topics[2])
		if err != nil {
			continue
		}
		toB58, err := evm20ToTronBase58(to20)
		if err != nil {
			continue
		}
		if toB58 != wantTo {
			continue
		}

		// ดึง amount จาก data
		amt, err := uint256AmountDecimal(lg.Data, USDTDecimals)
		if err != nil {
			continue
		}

		return TxObservation{
			TxHash:        txHash,
			ToAddress:     wantTo,
			Amount:        amt,
			Confirmations: confs,
			Success:       success,
			TokenContract: contractB58,
			BlockNumber:   txInfo.BlockNumber,
			BlockTime:     time.UnixMilli(txInfo.BlockTimestamp),
		}, nil
	}

	return TxObservation{}, fmt.Errorf("no matching USDT transfer log")
}

// RPC calls
func (c *Client) getTxInfo(ctx context.Context, baseURL string, useKey bool, txHash string) (*txInfoResp, error) {
	var resp txInfoResp
	err := c.postJSON(ctx, baseURL, "/wallet/gettransactioninfobyid", useKey, 
		map[string]string{"value": txHash}, &resp)
	if err != nil {
		return nil, err
	}
	return &resp, nil
}

func (c *Client) getNowBlock(ctx context.Context, baseURL string, useKey bool) (int64, error) {
	var resp nowBlockResp
	err := c.getJSON(ctx, baseURL, "/wallet/getnowblock", useKey, &resp)
	if err != nil {
		return 0, err
	}
	return resp.BlockHeader.RawData.Number, nil
}

// HTTP helpers
func (c *Client) postJSON(ctx context.Context, baseURL, path string, useKey bool, in, out interface{}) error {
	b, _ := json.Marshal(in)
	req, _ := http.NewRequestWithContext(ctx, http.MethodPost, baseURL+path, bytes.NewReader(b))
	req.Header.Set("Content-Type", "application/json")
	req.Header.Set("Accept", "application/json")
	if useKey && c.APIKey != "" {
		req.Header.Set("TRON-PRO-API-KEY", c.APIKey)
	}

	resp, err := c.HTTP.Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()

	if resp.StatusCode == 404 {
		return fmt.Errorf("not found")
	}
	if resp.StatusCode != 200 {
		return fmt.Errorf("unexpected status: %d", resp.StatusCode)
	}
	return json.NewDecoder(resp.Body).Decode(out)
}

func (c *Client) getJSON(ctx context.Context, baseURL, path string, useKey bool, out interface{}) error {
	req, _ := http.NewRequestWithContext(ctx, http.MethodGet, baseURL+path, nil)
	req.Header.Set("Accept", "application/json")
	if useKey && c.APIKey != "" {
		req.Header.Set("TRON-PRO-API-KEY", c.APIKey)
	}

	resp, err := c.HTTP.Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()

	if resp.StatusCode != 200 {
		return fmt.Errorf("unexpected status: %d", resp.StatusCode)
	}
	return json.NewDecoder(resp.Body).Decode(out)
}

// Address helpers
func normalizeBase58(addrStr string) (string, error) {
	a := strings.TrimSpace(addrStr)
	if a == "" {
		return "", fmt.Errorf("empty address")
	}
	if strings.HasPrefix(a, "T") {
		if !address.IsAddress(a) {
			return "", fmt.Errorf("invalid base58 address: %s", a)
		}
		return a, nil
	}
	// hex (41...) maybe with 0x
	h := strings.TrimPrefix(strings.TrimPrefix(a, "0x"), "0X")
	b58, err := address.HexToBase58(h)
	if err != nil {
		return "", err
	}
	if !address.IsAddress(b58) {
		return "", fmt.Errorf("invalid address after hex->base58: %s", b58)
	}
	return b58, nil
}

func evm20ToTronBase58(evm20 []byte) (string, error) {
	if len(evm20) != 20 {
		return "", fmt.Errorf("evm20 len %d", len(evm20))
	}
	h := "41" + hex.EncodeToString(evm20)
	return normalizeBase58(h)
}

func tronContractHexToBase58(addrHex string) (string, error) {
	h := strings.TrimPrefix(strings.TrimPrefix(addrHex, "0x"), "0X")
	h = strings.ToLower(h)
	if len(h) == 40 {
		h = "41" + h
	}
	if !strings.HasPrefix(h, "41") || len(h) != 42 {
		return "", fmt.Errorf("unexpected tron contract hex: %s", addrHex)
	}
	return normalizeBase58(h)
}

func last20BytesFromTopic(topic string) ([]byte, error) {
	t := strings.TrimPrefix(topic, "0x")
	b, err := hex.DecodeString(t)
	if err != nil {
		return nil, err
	}
	if len(b) != 32 {
		return nil, fmt.Errorf("topic len %d", len(b))
	}
	return b[12:], nil // last 20 bytes
}

func uint256AmountDecimal(data string, decimals int32) (decimal.Decimal, error) {
	d := strings.TrimPrefix(data, "0x")
	b, err := hex.DecodeString(d)
	if err != nil {
		return decimal.Zero, err
	}
	if len(b) != 32 {
		return decimal.Zero, fmt.Errorf("data len %d", len(b))
	}
	bi := new(big.Int).SetBytes(b)
	return decimal.NewFromBigInt(bi, 0).Shift(-decimals), nil
}

func transferTopic0() string {
	h := sha3.NewLegacyKeccak256()
	h.Write([]byte("Transfer(address,address,uint256)"))
	return "0x" + hex.EncodeToString(h.Sum(nil))
}
```

## **3. Database Migrations (db/migrations.go)**
```go
package db

const MigrationSQL = `
-- Users table
CREATE TABLE IF NOT EXISTS users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(20) DEFAULT 'customer',
    wallet_address VARCHAR(42),
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now()
);

-- Products/Services table
CREATE TABLE IF NOT EXISTS products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(30,6) NOT NULL,
    currency VARCHAR(10) DEFAULT 'USDT',
    network VARCHAR(10) NOT NULL,
    contract_address VARCHAR(42),
    status VARCHAR(20) DEFAULT 'active',
    created_at TIMESTAMPTZ DEFAULT now()
);

-- Payment Intents
CREATE TABLE IF NOT EXISTS payment_intents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    product_id UUID REFERENCES products(id),
    amount DECIMAL(30,6) NOT NULL,
    currency VARCHAR(10) DEFAULT 'USDT',
    network VARCHAR(10) NOT NULL,
    wallet_address VARCHAR(42) NOT NULL,
    status VARCHAR(30) DEFAULT 'pending',
    approval_status VARCHAR(20) DEFAULT 'none',
    underpaid_amount DECIMAL(30,6),
    paid_amount DECIMAL(30,6) DEFAULT 0,
    confirmation_target INT DEFAULT 12,
    expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now(),
    CONSTRAINT valid_status CHECK (status IN (
        'pending', 'succeeded', 'failed', 'expired', 
        'underpaid_pending_approval', 'refunding', 'refunded'
    )),
    CONSTRAINT valid_approval CHECK (approval_status IN (
        'none', 'pending', 'manual_approved', 'rejected'
    ))
);

-- Payment Transactions
CREATE TABLE IF NOT EXISTS payment_transactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payment_intent_id UUID REFERENCES payment_intents(id),
    tx_hash VARCHAR(128) NOT NULL,
    network VARCHAR(10) NOT NULL,
    from_address VARCHAR(42),
    to_address VARCHAR(42),
    amount DECIMAL(30,6) NOT NULL,
    confirmations INT DEFAULT 0,
    block_number BIGINT,
    block_time TIMESTAMPTZ,
    is_valid BOOLEAN DEFAULT false,
    validation_error TEXT,
    last_checked_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT now(),
    UNIQUE(tx_hash, network)
);

-- Underpaid Approvals
CREATE TABLE IF NOT EXISTS underpaid_approvals (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payment_intent_id UUID NOT NULL REFERENCES payment_intents(id),
    expected_amount DECIMAL(30,6) NOT NULL,
    actual_amount DECIMAL(30,6) NOT NULL,
    short_amount DECIMAL(30,6) NOT NULL,
    tx_hash VARCHAR(128),
    wallet_address VARCHAR(42),
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMPTZ DEFAULT now(),
    approved_at TIMESTAMPTZ,
    approved_by VARCHAR(100),
    notes TEXT,
    CONSTRAINT check_status CHECK (status IN ('pending', 'approved', 'rejected'))
);

-- Indexes
CREATE INDEX IF NOT EXISTS idx_payment_intents_user ON payment_intents(user_id);
CREATE INDEX IF NOT EXISTS idx_payment_intents_status ON payment_intents(status);
CREATE INDEX IF NOT EXISTS idx_payment_transactions_intent ON payment_transactions(payment_intent_id);
CREATE INDEX IF NOT EXISTS idx_payment_transactions_txhash ON payment_transactions(tx_hash);
CREATE INDEX IF NOT EXISTS idx_underpaid_approvals_intent ON underpaid_approvals(payment_intent_id);
`
```

## **4. Worker Reconciler (worker/reconciler.go)**
```go
package worker

import (
	"context"
	"database/sql"
	"fmt"
	"log"
	"time"

	"payment-system/internal/config"
	"payment-system/internal/tron"
	"payment-system/internal/notification"

	"github.com/shopspring/decimal"
)

type Reconciler struct {
	DB           *sql.DB
	TronClient   *tron.Client
	Config       *config.Config
	AlertService *notification.AlertService
}

type TxJob struct {
	ID              string
	PaymentIntentID string
	TxHash          string
	Network         string
	TargetAddress   string
	TargetAmount    decimal.Decimal
	ConfTarget      int
	Attempts        int
}

func NewReconciler(db *sql.DB, cfg *config.Config) *Reconciler {
	return &Reconciler{
		DB:           db,
		TronClient:   tron.New(cfg.TronAPIKey, cfg.TronPrimaryURL, cfg.TronFallbackURL),
		Config:       cfg,
		AlertService: notification.NewAlertService(cfg),
	}
}

func (r *Reconciler) Start(ctx context.Context) error {
	log.Println("Starting payment reconciler worker...")

	ticker := time.NewTicker(r.Config.WorkerInterval)
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			return ctx.Err()
		case <-ticker.C:
			if err := r.processJobs(ctx); err != nil {
				log.Printf("Error processing jobs: %v", err)
			}
		}
	}
}

func (r *Reconciler) processJobs(ctx context.Context) error {
	jobs, err := r.fetchPendingJobs(ctx)
	if err != nil {
		return fmt.Errorf("fetch pending jobs: %w", err)
	}

	for _, job := range jobs {
		if err := r.processJob(ctx, job); err != nil {
			log.Printf("Error processing job %s: %v", job.TxHash, err)
		}
	}
	return nil
}

func (r *Reconciler) fetchPendingJobs(ctx context.Context) ([]TxJob, error) {
	rows, err := r.DB.QueryContext(ctx, `
		SELECT 
			pt.id,
			pt.payment_intent_id,
			pt.tx_hash,
			pt.network,
			pi.wallet_address as target_address,
			pi.amount as target_amount,
			pi.confirmation_target,
			COALESCE(pt.last_checked_at IS NOT NULL AND pt.last_checked_at > now() - interval '5 minutes', false) as recent_check
		FROM payment_transactions pt
		JOIN payment_intents pi ON pi.id = pt.payment_intent_id
		WHERE 
			(pt.is_valid = false OR pt.is_valid IS NULL)
			AND pt.validation_error IS NULL
			AND (pt.last_checked_at IS NULL OR pt.last_checked_at < now() - interval '1 minute')
			AND pi.status NOT IN ('succeeded', 'failed', 'expired')
		ORDER BY pt.created_at ASC
		LIMIT 50
	`)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var jobs []TxJob
	for rows.Next() {
		var job TxJob
		if err := rows.Scan(
			&job.ID, &job.PaymentIntentID, &job.TxHash, &job.Network,
			&job.TargetAddress, &job.TargetAmount, &job.ConfTarget,
		); err != nil {
			return nil, err
		}
		jobs = append(jobs, job)
	}
	return jobs, nil
}

func (r *Reconciler) processJob(ctx context.Context, job TxJob) error {
	switch job.Network {
	case "TRC20":
		return r.processTRC20Job(ctx, job)
	case "ERC20":
		return r.processERC20Job(ctx, job)
	default:
		return r.failJob(ctx, job, "unsupported_network")
	}
}

func (r *Reconciler) processTRC20Job(ctx context.Context, job TxJob) error {
	obs, err := r.TronClient.ObserveUSDTTransferTo(ctx, job.TxHash, job.TargetAddress)
	if err != nil {
		return r.handleObservationError(ctx, job, err)
	}

	return r.applyObservation(ctx, job, obs)
}

func (r *Reconciler) applyObservation(ctx context.Context, job TxJob, obs tron.TxObservation) error {
	valid := false
	var validationErr string

	// ตรวจสอบความถูกต้อง
	if obs.Success && obs.Amount.GreaterThan(decimal.Zero) && obs.Confirmations >= int64(job.ConfTarget) {
		if obs.ToAddress == job.TargetAddress {
			valid = true
		} else {
			validationErr = "address_mismatch"
		}
	} else {
		if !obs.Success {
			validationErr = "tx_failed"
		} else if obs.Confirmations < int64(job.ConfTarget) {
			validationErr = "low_confirmations"
		} else if obs.Amount.LessThanOrEqual(decimal.Zero) {
			validationErr = "zero_amount"
		}
	}

	// อัปเดต transaction
	_, err := r.DB.ExecContext(ctx, `
		UPDATE payment_transactions 
		SET 
			to_address = $2,
			amount = $3,
			confirmations = $4,
			block_number = $5,
			block_time = $6,
			is_valid = $7,
			validation_error = CASE WHEN $7 THEN NULL ELSE $8 END,
			last_checked_at = now()
		WHERE id = $1
	`, job.ID, obs.ToAddress, obs.Amount.String(), obs.Confirmations,
		obs.BlockNumber, obs.BlockTime, valid, validationErr)

	if err != nil {
		return fmt.Errorf("update transaction: %w", err)
	}

	// ถ้า valid ให้สรุป intent
	if valid {
		return r.finalizeIntent(ctx, job)
	}

	return nil
}

func (r *Reconciler) finalizeIntent(ctx context.Context, job TxJob) error {
	// ดึงยอดรวมที่ชำระแล้ว
	var totalPaid decimal.Decimal
	err := r.DB.QueryRowContext(ctx, `
		SELECT COALESCE(SUM(amount), 0) 
		FROM payment_transactions 
		WHERE payment_intent_id = $1 AND is_valid = true
	`, job.PaymentIntentID).Scan(&totalPaid)
	if err != nil {
		return fmt.Errorf("get total paid: %w", err)
	}

	// ตรวจสอบยอด
	if totalPaid.GreaterThanOrEqual(job.TargetAmount) {
		// จ่ายครบ
		_, err = r.DB.ExecContext(ctx, `
			UPDATE payment_intents 
			SET status = 'succeeded', 
				paid_amount = $2,
				updated_at = now()
			WHERE id = $1
		`, job.PaymentIntentID, totalPaid.String())
		
		// ส่งอีเมลยืนยัน
		go r.AlertService.SendPaymentSuccess(job.PaymentIntentID)
		
	} else if totalPaid.GreaterThan(decimal.Zero) && totalPaid.LessThan(job.TargetAmount) {
		// จ่ายไม่ครบ - รออนุมัติจากบอส
		shortAmount := job.TargetAmount.Sub(totalPaid)
		
		_, err = r.DB.ExecContext(ctx, `
			UPDATE payment_intents 
			SET status = 'underpaid_pending_approval',
				approval_status = 'pending',
				underpaid_amount = $2,
				paid_amount = $3,
				updated_at = now()
			WHERE id = $1
		`, job.PaymentIntentID, shortAmount.String(), totalPaid.String())

		// ส่งแจ้งเตือนไปยังแอดมิน
		go r.AlertService.SendUnderpaidAlert(job.PaymentIntentID, job.TargetAmount, totalPaid)
	}

	return err
}

func (r *Reconciler) handleObservationError(ctx context.Context, job TxJob, err error) error {
	errorStr := err.Error()
	
	// ถ้าเป็น error ชั่วคราว ให้ retry
	if errorStr == "not found" || errorStr == "tx not found yet" {
		return nil // จะ retry ในรอบถัดไป
	}

	// error ถาวร ให้ fail job
	return r.failJob(ctx, job, errorStr)
}

func (r *Reconciler) failJob(ctx context.Context, job TxJob, reason string) error {
	_, err := r.DB.ExecContext(ctx, `
		UPDATE payment_transactions 
		SET is_valid = false, 
			validation_error = $2,
			last_checked_at = now()
		WHERE id = $1
	`, job.ID, reason)
	return err
}
```

## **5. Admin Handler (admin/handler.go)**
```go
package admin

import (
	"database/sql"
	"encoding/json"
	"net/http"
	"time"

	"payment-system/internal/config"
	"payment-system/internal/notification"
)

type AdminHandler struct {
	DB           *sql.DB
	Config       *config.Config
	AlertService *notification.AlertService
}

func NewAdminHandler(db *sql.DB, cfg *config.Config) *AdminHandler {
	return &AdminHandler{
		DB:           db,
		Config:       cfg,
		AlertService: notification.NewAlertService(cfg),
	}
}

type UnderpaidRequest struct {
	IntentID string `json:"intent_id"`
	Action   string `json:"action"` // "approve" or "reject"
	Notes    string `json:"notes"`
}

func (h *AdminHandler) HandleUnderpaidApproval(w http.ResponseWriter, r *http.Request) {
	var req UnderpaidRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "Invalid request", http.StatusBadRequest)
		return
	}

	// เริ่ม transaction
	tx, err := h.DB.Begin()
	if err != nil {
		http.Error(w, "Database error", http.StatusInternalServerError)
		return
	}
	defer tx.Rollback()

	switch req.Action {
	case "approve":
		// อัปเดต underpaid_approvals
		_, err = tx.Exec(`
			UPDATE underpaid_approvals 
			SET status = 'approved', 
				approved_at = now(), 
				approved_by = $2,
				notes = $3
			WHERE payment_intent_id = $1 AND status = 'pending'
		`, req.IntentID, r.Header.Get("X-User-ID"), req.Notes)

		// อัปเดต payment_intent
		_, err = tx.Exec(`
			UPDATE payment_intents 
			SET status = 'succeeded', 
				approval_status = 'manual_approved',
				updated_at = now()
			WHERE id = $1
		`, req.IntentID)

	case "reject":
		_, err = tx.Exec(`
			UPDATE underpaid_approvals 
			SET status = 'rejected', 
				approved_at = now(), 
				approved_by = $2,
				notes = $3
			WHERE payment_intent_id = $1 AND status = 'pending'
		`, req.IntentID, r.Header.Get("X-User-ID"), req.Notes)

		_, err = tx.Exec(`
			UPDATE payment_intents 
			SET status = 'pending', 
				approval_status = 'rejected',
				updated_at = now()
			WHERE id = $1
		`, req.IntentID)
	}

	if err != nil {
		http.Error(w, "Update failed", http.StatusInternalServerError)
		return
	}

	if err := tx.Commit(); err != nil {
		http.Error(w, "Commit failed", http.StatusInternalServerError)
		return
	}

	w.WriteHeader(http.StatusOK)
	json.NewEncoder(w).Encode(map[string]string{"status": "success"})
}

func (h *AdminHandler) GetPendingApprovals(w http.ResponseWriter, r *http.Request) {
	rows, err := h.DB.Query(`
		SELECT 
			ua.id,
			ua.payment_intent_id,
			ua.expected_amount,
			ua.actual_amount,
			ua.short_amount,
			ua.tx_hash,
			ua.wallet_address,
			ua.created_at,
			pi.status as intent_status
		FROM underpaid_approvals ua
		JOIN payment_intents pi ON pi.id = ua.payment_intent_id
		WHERE ua.status = 'pending'
		ORDER BY ua.created_at DESC
	`)
	if err != nil {
		http.Error(w, "Query failed", http.StatusInternalServerError)
		return
	}
	defer rows.Close()

	type Approval struct {
		ID             string    `json:"id"`
		IntentID       string    `json:"intent_id"`
		ExpectedAmount string    `json:"expected_amount"`
		ActualAmount   string    `json:"actual_amount"`
		ShortAmount    string    `json:"short_amount"`
		TxHash         string    `json:"tx_hash"`
		WalletAddress  string    `json:"wallet_address"`
		CreatedAt      time.Time `json:"created_at"`
		Status         string    `json:"status"`
	}

	var approvals []Approval
	for rows.Next() {
		var a Approval
		rows.Scan(&a.ID, &a.IntentID, &a.ExpectedAmount, &a.ActualAmount, 
			&a.ShortAmount, &a.TxHash, &a.WalletAddress, &a.CreatedAt, &a.Status)
		approvals = append(approvals, a)
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(approvals)
}
```

## **6. Notification Service (notification/alert.go)**
```go
package notification

import (
	"bytes"
	"fmt"
	"log"
	"net/smtp"
	"text/template"

	"payment-system/internal/config"

	"github.com/shopspring/decimal"
)

type AlertService struct {
	Config *config.Config
}

func NewAlertService(cfg *config.Config) *AlertService {
	return &AlertService{
		Config: cfg,
	}
}

func (s *AlertService) SendUnderpaidAlert(intentID string, expected, actual decimal.Decimal) {
	short := expected.Sub(actual)

	// Slack notification
	if s.Config.SlackWebhookURL != "" {
		message := fmt.Sprintf(`
🚨 *แจ้งเตือนยอดเงินไม่ครบ* 🚨

📋 Intent ID: %s
💰 ยอดที่คาดหวัง: %s USDT
💵 ยอดที่ได้รับ: %s USDT
📉 ยอดขาด: %s USDT

🔗 อนุมัติ: http://%s/admin/approve/%s
❌ ปฏิเสธ: http://%s/admin/reject/%s
`, intentID, expected.String(), actual.String(), short.String(),
			s.Config.AdminPort, intentID, s.Config.AdminPort, intentID)

		s.sendSlack(message)
	}

	// Email notification
	if s.Config.SMTPHost != "" {
		subject := fmt.Sprintf("🚨 ต้องการอนุมัติ: ยอดชำระไม่ครบ %s USDT", short.String())
		body := fmt.Sprintf(`
<html>
<body>
<h2>แจ้งเตือนยอดเงินไม่ครบ</h2>
<p>Intent ID: %s</p>
<p>ยอดที่คาดหวัง: %s USDT</p>
<p>ยอดที่ได้รับ: %s USDT</p>
<p>ยอดขาด: %s USDT</p>
<a href="http://%s/admin/approve/%s">✅ อนุมัติ</a>
<a href="http://%s/admin/reject/%s">❌ ปฏิเสธ</a>
</body>
</html>
`, intentID, expected.String(), actual.String(), short.String(),
			s.Config.Admin
