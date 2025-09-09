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



CREATE OR ALTER TRIGGER trg_DividendsPaid_Delete
ON DividendsPaid
AFTER UPDATE
AS
BEGIN
    INSERT INTO DividendsPaidAudit
    (
        DividendPaidId, Action, CorrelationId,
        AffiliateId, EffectiveDate, Amount, Currency,
        EquityClassId, PercentageOwned, IsSurplusElection,
        IsCapitalElection, Active, CreatedBy, CreatedOn
    )
    SELECT 
        i.Id, 'Remove', NEWID(),   -- fresh GUID for delete
        i.AffiliateId, i.EffectiveDate, i.Amount, i.Currency,
        i.EquityClassId, i.PercentageOwned, i.IsSurplusElection,
        i.IsCapitalElection, i.Active, i.UpdatedBy, GETUTCDATE()
    FROM inserted i
    INNER JOIN deleted d ON i.Id = d.Id
    WHERE d.Active = 1 AND i.Active = 0;
END;




CREATE OR ALTER TRIGGER trg_DividendsPaid_Insert
ON DividendsPaid
AFTER INSERT
AS
BEGIN
    INSERT INTO DividendsPaidAudit
    (
        DividendPaidId, Action, CorrelationId,
        AffiliateId, EffectiveDate, Amount, Currency,
        EquityClassId, PercentageOwned, IsSurplusElection,
        IsCapitalElection, Active, CreatedBy, CreatedOn
    )
    SELECT 
        i.Id, 'Insert', NEWID(),   -- new GUID every time
        i.AffiliateId, i.EffectiveDate, i.Amount, i.Currency,
        i.EquityClassId, i.PercentageOwned, i.IsSurplusElection,
        i.IsCapitalElection, i.Active, i.CreatedBy, GETUTCDATE()
    FROM inserted i;
END;




CREATE OR ALTER TRIGGER trg_DividendsPaid_Update
ON DividendsPaid
AFTER UPDATE
AS
BEGIN
    INSERT INTO DividendsPaidAudit
    (
        DividendPaidId, Action, CorrelationId,
        AffiliateId, EffectiveDate, Amount, Currency,
        EquityClassId, PercentageOwned, IsSurplusElection,
        IsCapitalElection, Active, CreatedBy, CreatedOn
    )
    SELECT 
        i.Id, 'Update', NEWID(),   -- new GUID on each update
        i.AffiliateId, i.EffectiveDate, i.Amount, i.Currency,
        i.EquityClassId, i.PercentageOwned, i.IsSurplusElection,
        i.IsCapitalElection, i.Active, i.UpdatedBy, GETUTCDATE()
    FROM inserted i;
END;
