-- Main Table
CREATE TABLE DividendsPaid (
    Id INT IDENTITY(1,1) PRIMARY KEY,
    AffiliateId INT NOT NULL,
    EffectiveDate DATETIME NOT NULL,
    Amount DECIMAL(18,2) NOT NULL,
    Currency VARCHAR(5) NOT NULL,
    EquityClassId INT NOT NULL,
    PercentageOwned DECIMAL(28,12) NOT NULL,
    IsSurplusElection BIT NOT NULL,
    IsCapitalElection BIT NOT NULL,
    Active BIT NOT NULL, -- 1 = Active, 0 = Soft Deleted
    CreatedBy NVARCHAR(250) NOT NULL,
    CreatedOn DATETIME NOT NULL DEFAULT(GETUTCDATE()),
    UpdatedBy NVARCHAR(250) NULL,
    UpdatedOn DATETIME NULL,
    CorrelationId UNIQUEIDENTIFIER NOT NULL DEFAULT NEWID()
);

-- Audit Table
CREATE TABLE DividendsPaidAudit (
    Id INT IDENTITY(1,1) PRIMARY KEY,
    DividendPaidId INT NOT NULL,
    Action VARCHAR(10) NOT NULL,
    CorrelationId UNIQUEIDENTIFIER NOT NULL,
    AffiliateId INT NOT NULL,
    EffectiveDate DATETIME NOT NULL,
    Amount DECIMAL(18,2) NOT NULL,
    Currency VARCHAR(5) NOT NULL,
    EquityClassId INT NOT NULL,
    PercentageOwned DECIMAL(28,12) NOT NULL,
    IsSurplusElection BIT NOT NULL,
    IsCapitalElection BIT NOT NULL,
    Active BIT NOT NULL,
    CreatedBy NVARCHAR(250) NOT NULL,
    CreatedOn DATETIME NOT NULL DEFAULT(GETUTCDATE())
);



CREATE OR ALTER TRIGGER trg_DividendsPaid_Audit
ON DividendsPaid
AFTER INSERT, UPDATE
AS
BEGIN
    SET NOCOUNT ON;

    ----------------------------------------------------------------
    -- 1. Handle Inserts
    ----------------------------------------------------------------
    INSERT INTO DividendsPaidAudit
    (
        DividendPaidId, Action, CorrelationId,
        AffiliateId, EffectiveDate, Amount, Currency,
        EquityClassId, PercentageOwned, IsSurplusElection,
        IsCapitalElection, Active, CreatedBy, CreatedOn
    )
    SELECT 
        i.Id, 'Insert', NEWID(),
        i.AffiliateId, i.EffectiveDate, i.Amount, i.Currency,
        i.EquityClassId, i.PercentageOwned, i.IsSurplusElection,
        i.IsCapitalElection, i.Active, 
        ISNULL(i.CreatedBy, 'System'), GETUTCDATE()
    FROM inserted i
    LEFT JOIN deleted d ON i.Id = d.Id
    WHERE d.Id IS NULL;
    ----------------------------------------------------------------


    ----------------------------------------------------------------
    -- 2. Handle Updates (Before/After logging)
    ----------------------------------------------------------------
    -- Before Update
    INSERT INTO DividendsPaidAudit
    (
        DividendPaidId, Action, CorrelationId,
        AffiliateId, EffectiveDate, Amount, Currency,
        EquityClassId, PercentageOwned, IsSurplusElection,
        IsCapitalElection, Active, CreatedBy, CreatedOn
    )
    SELECT 
        d.Id, 'Update-BF', NEWID(),
        d.AffiliateId, d.EffectiveDate, d.Amount, d.Currency,
        d.EquityClassId, d.PercentageOwned, d.IsSurplusElection,
        d.IsCapitalElection, d.Active, 
        ISNULL(d.UpdatedBy, ISNULL(d.CreatedBy, 'System')), GETUTCDATE()
    FROM deleted d
    INNER JOIN inserted i ON d.Id = i.Id
    WHERE 
        (
            ISNULL(d.AffiliateId, -1) <> ISNULL(i.AffiliateId, -1) OR
            ISNULL(d.EffectiveDate, '1900-01-01') <> ISNULL(i.EffectiveDate, '1900-01-01') OR
            ISNULL(d.Amount, -1) <> ISNULL(i.Amount, -1) OR
            ISNULL(d.Currency, '') <> ISNULL(i.Currency, '') OR
            ISNULL(d.EquityClassId, -1) <> ISNULL(i.EquityClassId, -1) OR
            ISNULL(d.PercentageOwned, -1) <> ISNULL(i.PercentageOwned, -1) OR
            ISNULL(d.IsSurplusElection, -1) <> ISNULL(i.IsSurplusElection, -1) OR
            ISNULL(d.IsCapitalElection, -1) <> ISNULL(i.IsCapitalElection, -1) OR
            ISNULL(d.Active, -1) <> ISNULL(i.Active, -1)
        )
        AND NOT (d.Active = 1 AND i.Active = 0);

    -- After Update
    INSERT INTO DividendsPaidAudit
    (
        DividendPaidId, Action, CorrelationId,
        AffiliateId, EffectiveDate, Amount, Currency,
        EquityClassId, PercentageOwned, IsSurplusElection,
        IsCapitalElection, Active, CreatedBy, CreatedOn
    )
    SELECT 
        i.Id, 'Update-AF', NEWID(),
        i.AffiliateId, i.EffectiveDate, i.Amount, i.Currency,
        i.EquityClassId, i.PercentageOwned, i.IsSurplusElection,
        i.IsCapitalElection, i.Active, 
        ISNULL(i.UpdatedBy, ISNULL(i.CreatedBy, 'System')), GETUTCDATE()
    FROM inserted i
    INNER JOIN deleted d ON d.Id = i.Id
    WHERE 
        (
            ISNULL(d.AffiliateId, -1) <> ISNULL(i.AffiliateId, -1) OR
            ISNULL(d.EffectiveDate, '1900-01-01') <> ISNULL(i.EffectiveDate, '1900-01-01') OR
            ISNULL(d.Amount, -1) <> ISNULL(i.Amount, -1) OR
            ISNULL(d.Currency, '') <> ISNULL(i.Currency, '') OR
            ISNULL(d.EquityClassId, -1) <> ISNULL(i.EquityClassId, -1) OR
            ISNULL(d.PercentageOwned, -1) <> ISNULL(i.PercentageOwned, -1) OR
            ISNULL(d.IsSurplusElection, -1) <> ISNULL(i.IsSurplusElection, -1) OR
            ISNULL(d.IsCapitalElection, -1) <> ISNULL(i.IsCapitalElection, -1) OR
            ISNULL(d.Active, -1) <> ISNULL(i.Active, -1)
        )
        AND NOT (d.Active = 1 AND i.Active = 0);
    ----------------------------------------------------------------


    ----------------------------------------------------------------
    -- 3. Handle Soft Deletes (Active = 1 → Active = 0)
    ----------------------------------------------------------------
    INSERT INTO DividendsPaidAudit
    (
        DividendPaidId, Action, CorrelationId,
        AffiliateId, EffectiveDate, Amount, Currency,
        EquityClassId, PercentageOwned, IsSurplusElection,
        IsCapitalElection, Active, CreatedBy, CreatedOn
    )
    SELECT 
        i.Id, 'Remove', NEWID(),
        i.AffiliateId, i.EffectiveDate, i.Amount, i.Currency,
        i.EquityClassId, i.PercentageOwned, i.IsSurplusElection,
        i.IsCapitalElection, i.Active, 
        ISNULL(i.UpdatedBy, ISNULL(i.CreatedBy, 'System')), GETUTCDATE()
    FROM inserted i
    INNER JOIN deleted d ON d.Id = i.Id
    WHERE d.Active = 1 AND i.Active = 0;
    ----------------------------------------------------------------
END;

