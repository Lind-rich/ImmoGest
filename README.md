# ImmoGest
Application de gestion des immeubles , chambres et locataires d'un propriétaire


-- ============================================================
--   BASE DE DONNÉES : ImmoGest — Système de Gestion de Logement
--   Version 2.0 — Données contextualisées pour le Cameroun
--   Ville : Douala | Monnaie : FCFA
-- ============================================================

CREATE DATABASE IF NOT EXISTS immogest
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

USE immogest;

-- ============================================================
--  1. TABLE : bailleur
-- ============================================================
CREATE TABLE bailleur (
    id_bailleur   INT AUTO_INCREMENT PRIMARY KEY,
    nom           VARCHAR(100)  NOT NULL,
    prenom        VARCHAR(100)  NOT NULL,
    email         VARCHAR(150)  NOT NULL UNIQUE,
    telephone     VARCHAR(25),
    mot_de_passe  VARCHAR(255)  NOT NULL,         -- hashé en bcrypt
    date_creation DATETIME      DEFAULT CURRENT_TIMESTAMP
);

-- ============================================================
--  2. TABLE : immeuble
-- ============================================================
CREATE TABLE immeuble (
    id_immeuble   INT AUTO_INCREMENT PRIMARY KEY,
    nom_immeuble  VARCHAR(150)  NOT NULL,           -- ex: "Résidence Belle Vue"
    adresse       VARCHAR(255)  NOT NULL,
    ville         VARCHAR(100)  NOT NULL,            -- ex: "Douala" ou "Yaoundé"
    quartier      VARCHAR(100)  NOT NULL,            -- ex: "Bonapriso", "Akwa", "Bonanjo"
    nombre_etage  INT           NOT NULL DEFAULT 1,
    superficie    DECIMAL(10,2),                     -- m²
    id_bailleur   INT           NOT NULL,
    CONSTRAINT fk_immeuble_bailleur
        FOREIGN KEY (id_bailleur) REFERENCES bailleur(id_bailleur)
        ON DELETE CASCADE ON UPDATE CASCADE
);

-- ============================================================
--  3. TABLE : etage
-- ============================================================
CREATE TABLE etage (
    id_etage      INT AUTO_INCREMENT PRIMARY KEY,
    numero        INT           NOT NULL,            -- 0 = RDC, 1, 2, 3...
    description   VARCHAR(255),
    id_immeuble   INT           NOT NULL,
    CONSTRAINT fk_etage_immeuble
        FOREIGN KEY (id_immeuble) REFERENCES immeuble(id_immeuble)
        ON DELETE CASCADE ON UPDATE CASCADE
);

-- ============================================================
--  4. TABLE : chambre
-- ============================================================
CREATE TABLE chambre (
    id_chambre    INT AUTO_INCREMENT PRIMARY KEY,
    numero        VARCHAR(20)   NOT NULL,
    type_chambre  ENUM('Studio','F1','F2','F3','F4','Chambre simple') NOT NULL DEFAULT 'Studio',
    statut        ENUM('libre','occupee','expiration_proche','bail_expire','maintenance')
                  NOT NULL DEFAULT 'libre',
    capacite      INT           NOT NULL DEFAULT 1,
    prix_loyer    DECIMAL(10,2) NOT NULL,            -- loyer mensuel en FCFA
    superficie    DECIMAL(8,2),                       -- m²
    description   VARCHAR(255),
    id_etage      INT           NOT NULL,
    CONSTRAINT fk_chambre_etage
        FOREIGN KEY (id_etage) REFERENCES etage(id_etage)
        ON DELETE CASCADE ON UPDATE CASCADE
);

-- ============================================================
--  5. TABLE : locataire
-- ============================================================
CREATE TABLE locataire (
    id_locataire        INT AUTO_INCREMENT PRIMARY KEY,
    prenom              VARCHAR(100)  NOT NULL,
    nom                 VARCHAR(100)  NOT NULL,
    email               VARCHAR(150)  NOT NULL UNIQUE,
    telephone           VARCHAR(25)   NOT NULL,       -- format: +237 6XX XX XX XX
    numero_piece_id     VARCHAR(100)  NOT NULL,        -- CNI camerounaise
    notes               TEXT,                          -- observations du bailleur
    date_enregistrement DATETIME      DEFAULT CURRENT_TIMESTAMP
);

-- ============================================================
--  6. TABLE : contrat
-- ============================================================
CREATE TABLE contrat (
    id_contrat              INT AUTO_INCREMENT PRIMARY KEY,
    date_debut              DATE          NOT NULL,
    date_fin                DATE          NOT NULL,
    loyer_mensuel           DECIMAL(10,2) NOT NULL,    -- en FCFA
    statut                  ENUM('en_attente','actif','expire','resilié','renouvele')
                            NOT NULL DEFAULT 'en_attente',
    nombre_prolongations    INT           NOT NULL DEFAULT 0,
    date_derniere_prolongation DATE,
    id_chambre              INT           NOT NULL,
    id_locataire            INT           NOT NULL,
    id_bailleur             INT           NOT NULL,
    date_creation           DATETIME      DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_contrat_chambre
        FOREIGN KEY (id_chambre)   REFERENCES chambre(id_chambre)   ON UPDATE CASCADE,
    CONSTRAINT fk_contrat_locataire
        FOREIGN KEY (id_locataire) REFERENCES locataire(id_locataire) ON UPDATE CASCADE,
    CONSTRAINT fk_contrat_bailleur
        FOREIGN KEY (id_bailleur)  REFERENCES bailleur(id_bailleur)   ON UPDATE CASCADE,
    CONSTRAINT chk_dates_contrat
        CHECK (date_fin > date_debut)
);

-- ============================================================
--  7. TABLE : historique_prolongation
--     Trace chaque prolongation de bail
-- ============================================================
CREATE TABLE historique_prolongation (
    id_prolongation   INT AUTO_INCREMENT PRIMARY KEY,
    id_contrat        INT           NOT NULL,
    ancienne_date_fin DATE          NOT NULL,
    nouvelle_date_fin DATE          NOT NULL,
    motif             VARCHAR(255),
    date_prolongation DATETIME      DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_prolongation_contrat
        FOREIGN KEY (id_contrat) REFERENCES contrat(id_contrat)
        ON DELETE CASCADE ON UPDATE CASCADE
);

-- ============================================================
--  8. TABLE : paiement
--     Gère les paiements mensuels (MTN, Orange, espèces, virement)
-- ============================================================
CREATE TABLE paiement (
    id_paiement       INT AUTO_INCREMENT PRIMARY KEY,
    id_contrat        INT              NOT NULL,
    montant           DECIMAL(10,2)    NOT NULL,        -- en FCFA
    date_paiement     DATE             NOT NULL,
    mois_concerne     VARCHAR(7)       NOT NULL,         -- format: "2026-02"
    mode_paiement     ENUM('especes','virement','MTN_Mobile_Money',
                           'Orange_Money','Express_Union','cheque')
                      NOT NULL DEFAULT 'especes',
    statut            ENUM('paye','en_retard','partiel','annule')
                      NOT NULL DEFAULT 'paye',
    reference         VARCHAR(100),                      -- numéro de transaction / reçu
    notes             TEXT,
    date_enregistrement DATETIME      DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_paiement_contrat
        FOREIGN KEY (id_contrat) REFERENCES contrat(id_contrat) ON UPDATE CASCADE,
    CONSTRAINT uq_paiement_mois UNIQUE (id_contrat, mois_concerne)
);

-- ============================================================
--  9. TABLE : mail_interne
--     Historique de tous les mails envoyés via l'API
--     Alimente l'écran Messagerie
-- ============================================================
CREATE TABLE mail_interne (
    id_mail            INT AUTO_INCREMENT PRIMARY KEY,
    type_mail          ENUM('CONTRAT','RAPPEL_BAIL','ALERTE_EXPIRATION',
                            'ALERTE_EXPIRE','PAIEMENT_RECU','MANUEL')
                       NOT NULL,
    destinataire_email VARCHAR(150)  NOT NULL,
    destinataire_nom   VARCHAR(200)  NOT NULL,
    sujet              VARCHAR(255)  NOT NULL,
    corps              TEXT          NOT NULL,
    statut_envoi       ENUM('envoye','echec','en_attente') NOT NULL DEFAULT 'en_attente',
    date_envoi         DATETIME      DEFAULT CURRENT_TIMESTAMP,
    id_contrat         INT,
    id_bailleur        INT,
    erreur_detail      TEXT,
    CONSTRAINT fk_mail_contrat
        FOREIGN KEY (id_contrat)  REFERENCES contrat(id_contrat)
        ON DELETE SET NULL ON UPDATE CASCADE,
    CONSTRAINT fk_mail_bailleur
        FOREIGN KEY (id_bailleur) REFERENCES bailleur(id_bailleur)
        ON DELETE SET NULL ON UPDATE CASCADE
);

-- ============================================================
--  10. TABLE : config_mail_api
--      Paramètres API mail stockés en base
-- ============================================================
CREATE TABLE config_mail_api (
    id_config          INT AUTO_INCREMENT PRIMARY KEY,
    fournisseur        ENUM('SendGrid','Brevo','JavaMail_Gmail','Autre') NOT NULL,
    api_key            VARCHAR(500)  NOT NULL,
    email_expediteur   VARCHAR(150)  NOT NULL,
    nom_expediteur     VARCHAR(150)  NOT NULL,
    delai_rappel_jours INT           NOT NULL DEFAULT 30,
    actif              BOOLEAN       NOT NULL DEFAULT TRUE,
    date_maj           DATETIME      DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- ============================================================
--  INDEXES
-- ============================================================
CREATE INDEX idx_chambre_statut      ON chambre(statut);
CREATE INDEX idx_contrat_statut      ON contrat(statut);
CREATE INDEX idx_contrat_date_fin    ON contrat(date_fin);
CREATE INDEX idx_contrat_locataire   ON contrat(id_locataire);
CREATE INDEX idx_contrat_bailleur    ON contrat(id_bailleur);
CREATE INDEX idx_paiement_contrat    ON paiement(id_contrat);
CREATE INDEX idx_paiement_statut     ON paiement(statut);
CREATE INDEX idx_paiement_mois       ON paiement(mois_concerne);
CREATE INDEX idx_mail_type           ON mail_interne(type_mail);
CREATE INDEX idx_mail_statut         ON mail_interne(statut_envoi);

-- ============================================================
--  VUES
-- ============================================================

-- Vue: Chambres disponibles
CREATE OR REPLACE VIEW v_chambres_disponibles AS
SELECT
    c.id_chambre,
    c.numero,
    c.type_chambre,
    c.prix_loyer,
    c.superficie,
    e.numero       AS numero_etage,
    i.nom_immeuble,
    i.quartier,
    i.ville,
    b.nom          AS nom_bailleur
FROM chambre c
JOIN etage    e ON c.id_etage    = e.id_etage
JOIN immeuble i ON e.id_immeuble = i.id_immeuble
JOIN bailleur b ON i.id_bailleur = b.id_bailleur
WHERE c.statut = 'libre';

-- Vue: Contrats actifs complets (écran Locataires)
CREATE OR REPLACE VIEW v_contrats_actifs AS
SELECT
    ct.id_contrat,
    ct.date_debut,
    ct.date_fin,
    ct.loyer_mensuel,
    ct.statut,
    ct.nombre_prolongations,
    DATEDIFF(ct.date_fin, CURDATE())    AS jours_restants,
    CONCAT(l.prenom, ' ', l.nom)        AS nom_locataire,
    l.email                             AS email_locataire,
    l.telephone,
    ch.numero                           AS numero_chambre,
    ch.type_chambre,
    i.nom_immeuble,
    i.quartier,
    i.ville,
    CONCAT(b.prenom, ' ', b.nom)        AS nom_bailleur,
    b.email                             AS email_bailleur
FROM contrat    ct
JOIN locataire  l  ON ct.id_locataire = l.id_locataire
JOIN chambre    ch ON ct.id_chambre   = ch.id_chambre
JOIN etage      e  ON ch.id_etage     = e.id_etage
JOIN immeuble   i  ON e.id_immeuble   = i.id_immeuble
JOIN bailleur   b  ON ct.id_bailleur  = b.id_bailleur
WHERE ct.statut IN ('actif','renouvele');

-- Vue: Paiements en retard (Dashboard)
CREATE OR REPLACE VIEW v_paiements_en_retard AS
SELECT
    p.id_paiement,
    p.mois_concerne,
    p.montant,
    p.statut,
    CONCAT(l.prenom, ' ', l.nom) AS nom_locataire,
    l.email,
    l.telephone,
    ch.numero                    AS numero_chambre,
    i.nom_immeuble,
    i.quartier
FROM paiement   p
JOIN contrat    ct ON p.id_contrat    = ct.id_contrat
JOIN locataire  l  ON ct.id_locataire = l.id_locataire
JOIN chambre    ch ON ct.id_chambre   = ch.id_chambre
JOIN etage      e  ON ch.id_etage     = e.id_etage
JOIN immeuble   i  ON e.id_immeuble   = i.id_immeuble
WHERE p.statut = 'en_retard';

-- Vue: Statistiques par immeuble (écran Rapports)
CREATE OR REPLACE VIEW v_stats_immeuble AS
SELECT
    i.id_immeuble,
    i.nom_immeuble,
    i.quartier,
    i.ville,
    COUNT(ch.id_chambre)                                                AS total_chambres,
    SUM(ch.statut = 'occupee')                                         AS chambres_occupees,
    SUM(ch.statut = 'libre')                                           AS chambres_libres,
    ROUND(SUM(ch.statut='occupee') * 100.0 / NULLIF(COUNT(ch.id_chambre),0), 1)
                                                                        AS taux_occupation,
    COALESCE(SUM(
        CASE WHEN ct.statut IN ('actif','renouvele') THEN ct.loyer_mensuel ELSE 0 END
    ), 0)                                                               AS revenus_mensuels,
    SUM(DATEDIFF(ct.date_fin, CURDATE()) BETWEEN 0 AND 30
        AND ct.statut = 'actif')                                       AS baux_expirant_bientot
FROM immeuble i
LEFT JOIN etage     e  ON e.id_immeuble  = i.id_immeuble
LEFT JOIN chambre   ch ON ch.id_etage    = e.id_etage
LEFT JOIN contrat   ct ON ct.id_chambre  = ch.id_chambre
                       AND ct.statut IN ('actif','renouvele')
GROUP BY i.id_immeuble, i.nom_immeuble, i.quartier, i.ville;

-- ============================================================
--  TRIGGERS
-- ============================================================
DELIMITER $$

-- Chambre → occupee à la création du contrat
CREATE TRIGGER trg_contrat_insert
AFTER INSERT ON contrat
FOR EACH ROW
BEGIN
    IF NEW.statut = 'actif' THEN
        UPDATE chambre SET statut = 'occupee' WHERE id_chambre = NEW.id_chambre;
    END IF;
END$$

-- Mise à jour statut chambre selon statut contrat
CREATE TRIGGER trg_contrat_update
AFTER UPDATE ON contrat
FOR EACH ROW
BEGIN
    IF NEW.statut = 'actif' AND OLD.statut != 'actif' THEN
        UPDATE chambre SET statut = 'occupee' WHERE id_chambre = NEW.id_chambre;
    END IF;
    IF NEW.statut IN ('resilié','expire') AND OLD.statut = 'actif' THEN
        UPDATE chambre SET statut = 'libre' WHERE id_chambre = NEW.id_chambre;
    END IF;
END$$

-- Historique automatique des prolongations
CREATE TRIGGER trg_prolongation_bail
AFTER UPDATE ON contrat
FOR EACH ROW
BEGIN
    IF NEW.date_fin != OLD.date_fin AND NEW.statut IN ('actif','renouvele') THEN
        INSERT INTO historique_prolongation
            (id_contrat, ancienne_date_fin, nouvelle_date_fin, motif)
        VALUES
            (NEW.id_contrat, OLD.date_fin, NEW.date_fin, 'Prolongation via ImmoGest');
    END IF;
END$$

DELIMITER ;

-- ============================================================
--  DONNÉES DE TEST — CONTEXTE CAMEROUN / DOUALA
-- ============================================================

-- Bailleur administrateur
INSERT INTO bailleur (nom, prenom, email, telephone, mot_de_passe) VALUES
('Tchouaméni', 'Jean-Daniel', 'jd.tchouameni@immogest.cm', '+237 6 70 12 34 56',
 '$2b$12$examplehashedpassword');

-- Immeubles à Douala (quartiers réels)
INSERT INTO immeuble (nom_immeuble, adresse, ville, quartier, nombre_etage, id_bailleur) VALUES
('Résidence Belle Vue',  'Rue de l\'Hôpital Laquintinie',   'Douala', 'Bonapriso',    3, 1),
('Immeuble Le Wouri',    'Avenue Ahmadou Ahidjo',            'Douala', 'Akwa',         2, 1),
('Résidence Colombe',    'Rue du Marché Central',            'Douala', 'Bonanjo',      4, 1),
('Villa des Flamboyants','Quartier Makepe Missoke, Bloc 7',  'Douala', 'Makepe',       2, 1);

-- Étages
INSERT INTO etage (numero, description, id_immeuble) VALUES
(0, 'Rez-de-chaussée', 1), (1, '1er étage', 1), (2, '2ème étage', 1),
(0, 'Rez-de-chaussée', 2), (1, '1er étage', 2),
(0, 'Rez-de-chaussée', 3), (1, '1er étage', 3),
(0, 'Rez-de-chaussée', 4);

-- Chambres (loyers réalistes en FCFA pour Douala)
INSERT INTO chambre (numero, type_chambre, statut, prix_loyer, superficie, id_etage) VALUES
('101', 'Studio',        'occupee',           45000, 18.0, 1),
('102', 'F1',            'libre',             60000, 28.0, 1),
('103', 'Studio',        'occupee',           45000, 18.0, 2),
('104', 'F2',            'expiration_proche', 80000, 45.0, 2),
('105', 'Chambre simple','bail_expire',        35000, 12.0, 3),
('201', 'F2',            'occupee',           85000, 48.0, 4),
('202', 'Studio',        'expiration_proche', 50000, 20.0, 5),
('301', 'F1',            'libre',             65000, 30.0, 6),
('302', 'F3',            'occupee',          110000, 65.0, 7),
('401', 'Studio',        'libre',             40000, 15.0, 8);

-- Locataires (noms camerounais)
INSERT INTO locataire (prenom, nom, email, telephone, numero_piece_id, notes) VALUES
('Aristide',   'Mvondo',     'a.mvondo@gmail.com',    '+237 6 70 11 11 11', 'CNI-CM-2024-100123', NULL),
('Karine',     'Belibi',     'k.belibi@gmail.com',    '+237 6 55 22 22 22', 'CNI-CM-2024-100456', 'Employée de banque'),
('Françoise',  'Ngono',      'f.ngono@yahoo.fr',      '+237 6 99 33 33 33', 'CNI-CM-2024-100789', NULL),
('Olivier',    'Atangana',   'o.atangana@gmail.com',  '+237 6 70 44 44 44', 'CNI-CM-2024-101012', 'Fonctionnaire'),
('Boris',      'Nkoulou',    'b.nkoulou@gmail.com',   '+237 6 55 55 55 55', 'CNI-CM-2024-101345', NULL),
('Patricia',   'Essomba',    'p.essomba@gmail.com',   '+237 6 77 66 66 66', 'CNI-CM-2024-101678', 'Commerçante au marché central');

-- Contrats
INSERT INTO contrat (date_debut, date_fin, loyer_mensuel, statut, id_chambre, id_locataire, id_bailleur) VALUES
('2025-10-01', '2026-10-01', 45000,  'actif',  1, 1, 1),
('2025-03-01', '2026-03-05', 80000,  'actif',  4, 2, 1),
('2025-08-01', '2026-08-01', 45000,  'actif',  3, 3, 1),
('2025-01-15', '2026-02-15', 35000,  'expire', 5, 4, 1),
('2025-12-01', '2026-12-01', 85000,  'actif',  6, 5, 1),
('2025-09-01', '2026-09-01', 110000, 'actif',  9, 6, 1);

-- Paiements (Mobile Money très utilisé au Cameroun)
INSERT INTO paiement (id_contrat, montant, date_paiement, mois_concerne, mode_paiement, statut, reference) VALUES
(1, 45000,  '2026-01-04', '2026-01', 'MTN_Mobile_Money',   'paye',      'MTN-20260104-7821'),
(1, 45000,  '2026-02-03', '2026-02', 'MTN_Mobile_Money',   'paye',      'MTN-20260203-8934'),
(2, 80000,  '2026-01-05', '2026-01', 'Orange_Money',       'paye',      'OM-20260105-4421'),
(2, 80000,  '2026-02-15', '2026-02', 'Orange_Money',       'en_retard', NULL),
(3, 45000,  '2026-01-01', '2026-01', 'especes',            'paye',      'ESP-20260101-001'),
(5, 85000,  '2026-01-02', '2026-01', 'virement',           'paye',      'VIR-20260102-3301'),
(5, 85000,  '2026-02-01', '2026-02', 'virement',           'paye',      'VIR-20260201-3402'),
(6, 110000, '2026-01-03', '2026-01', 'Express_Union',      'paye',      'EU-20260103-9901');

-- Config API mail — Brevo (conseillé pour le Cameroun)
INSERT INTO config_mail_api (fournisseur, api_key, email_expediteur, nom_expediteur, delai_rappel_jours) VALUES
('Brevo', 'VOTRE_CLE_API_BREVO_ICI', 'noreply@immogest.cm', 'ImmoGest Cameroun', 30);

-- Historique messagerie
INSERT INTO mail_interne (type_mail, destinataire_email, destinataire_nom, sujet, corps, statut_envoi, id_contrat, id_bailleur) VALUES
('CONTRAT',
 'a.mvondo@gmail.com', 'Aristide Mvondo',
 'Votre contrat de bail — Résidence Belle Vue, Chambre 101',
 'Bonjour Aristide, votre contrat de bail pour la chambre 101 (Résidence Belle Vue, Bonapriso) est disponible. Loyer : 45 000 FCFA/mois. Durée : 01/10/2025 au 01/10/2026.',
 'envoye', 1, 1),

('RAPPEL_BAIL',
 'k.belibi@gmail.com', 'Karine Belibi',
 'Rappel — Votre bail expire dans 7 jours',
 'Bonjour Karine, votre contrat de location (Chambre 104, Immeuble Le Wouri, Akwa) expire le 05/03/2026. Veuillez contacter votre bailleur pour un éventuel renouvellement.',
 'envoye', 2, 1),

('ALERTE_EXPIRE',
 'jd.tchouameni@immogest.cm', 'Jean-Daniel Tchouaméni',
 'Alerte : Bail expiré — Olivier Atangana — Ch.105',
 'Le contrat de bail de Olivier Atangana (Chambre 105, Résidence Colombe, Bonanjo) a expiré le 15/02/2026. Action requise.',
 'envoye', 4, 1),

('PAIEMENT_RECU',
 'a.mvondo@gmail.com', 'Aristide Mvondo',
 'Confirmation de paiement — Février 2026',
 'Bonjour Aristide, votre paiement de 45 000 FCFA pour le mois de Février 2026 a bien été reçu via MTN Mobile Money. Référence : MTN-20260203-8934.',
 'envoye', 1, 1);
