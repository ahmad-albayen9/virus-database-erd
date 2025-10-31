-- SQL DDL (Data Definition Language) for GiveHub Platform (PostgreSQL Syntax)
-- Version 2.0: Corrected and Optimized

-- 1. USERS Table (Base table for all accounts)
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL, -- Stored as a hash
    user_type VARCHAR(20) NOT NULL CHECK (user_type IN ('volunteer', 'charity', 'admin')),
    is_active BOOLEAN DEFAULT TRUE,
    -- Use TIMESTAMPTZ to store in UTC and handle timezones correctly
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- 2. VOLUNTEERS_PROFILE Table
CREATE TABLE volunteers_profile (
    user_id INTEGER PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    phone_number VARCHAR(20),
    city VARCHAR(100),
    bio TEXT,
    points INTEGER DEFAULT 0, -- For Gamification
    last_activity TIMESTAMP
);

-- 3. CHARITIES_PROFILE Table
CREATE TABLE charities_profile (
    user_id INTEGER PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(255) UNIQUE NOT NULL,
    license_number VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,
    website VARCHAR(255),
    contact_person VARCHAR(255),
    logo_url VARCHAR(255),
    is_verified BOOLEAN DEFAULT FALSE,
    registration_date DATE
);

-- 4. PROJECTS Table
CREATE TABLE projects (
    id SERIAL PRIMARY KEY,
    charity_id INTEGER NOT NULL REFERENCES charities_profile(user_id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL,
    description TEXT NOT NULL,
    start_date DATE,
    end_date DATE,
    status VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending', 'active', 'completed', 'cancelled')),
    required_volunteers INTEGER NOT NULL,
    location VARCHAR(255),
    created_at TIMESTAMP DEFAULT NOW()
);

-- 5. TEAMS Table (Teams within a Project)
-- Merged team_workspaces here for simplicity
CREATE TABLE teams (
    id SERIAL PRIMARY KEY,
    project_id INTEGER NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    max_members INTEGER NOT NULL,
    team_leader_id INTEGER REFERENCES volunteers_profile(user_id) ON DELETE SET NULL, -- Leader must be a volunteer
    workspace_channel_name VARCHAR(255) UNIQUE, -- For messaging channel, can be nullable
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE (project_id, name)
);

-- 6. TEAM_MEMBERS Table (Many-to-Many relationship between Volunteers and Teams)
CREATE TABLE team_members (
    volunteer_id INTEGER NOT NULL REFERENCES volunteers_profile(user_id) ON DELETE CASCADE,
    team_id INTEGER NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    joined_at TIMESTAMP DEFAULT NOW(),
    is_active BOOLEAN DEFAULT TRUE,
    PRIMARY KEY (volunteer_id, team_id)
);

-- 7. SKILLS_LOOKUP Table
CREATE TABLE skills_lookup (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL
);

-- 8. PROJECT_SKILLS Table (Skills required for a Project)
CREATE TABLE project_skills (
    project_id INTEGER NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    skill_id INTEGER NOT NULL REFERENCES skills_lookup(id) ON DELETE CASCADE,
    is_required BOOLEAN DEFAULT TRUE,
    PRIMARY KEY (project_id, skill_id)
);

-- 9. VOLUNTEER_SKILLS Table (Skills possessed by a Volunteer)
CREATE TABLE volunteer_skills (
    volunteer_id INTEGER NOT NULL REFERENCES volunteers_profile(user_id) ON DELETE CASCADE,
    skill_id INTEGER NOT NULL REFERENCES skills_lookup(id) ON DELETE CASCADE,
    proficiency_level INTEGER DEFAULT 1 CHECK (proficiency_level BETWEEN 1 AND 5),
    PRIMARY KEY (volunteer_id, skill_id)
);

-- 10. ACTIVITY_LOG Table (To track volunteer hours and points)
CREATE TABLE activity_log (
    id SERIAL PRIMARY KEY,
    volunteer_id INTEGER NOT NULL REFERENCES volunteers_profile(user_id) ON DELETE CASCADE,
    project_id INTEGER REFERENCES projects(id) ON DELETE SET NULL,
    team_id INTEGER REFERENCES teams(id) ON DELETE SET NULL,
    activity_type VARCHAR(50) NOT NULL CHECK (activity_type IN ('hours_logged', 'task_completed', 'training_attended')),
    duration_minutes INTEGER, -- Duration in minutes
    points_awarded INTEGER DEFAULT 0,
    log_date DATE NOT NULL,
    approved_by INTEGER REFERENCES users(id) ON DELETE SET NULL, -- Charity/Team Leader approval
    created_at TIMESTAMP DEFAULT NOW()
);

-- 11. RATINGS Table (360-degree feedback, Polymorphic Relation)
CREATE TABLE ratings (
    id SERIAL PRIMARY KEY,
    rater_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    rating_value INTEGER NOT NULL CHECK (rating_value BETWEEN 1 AND 5),
    comment TEXT,
    rated_entity_id INTEGER NOT NULL,
    rated_entity_type VARCHAR(20) NOT NULL CHECK (rated_entity_type IN ('volunteer', 'project', 'team', 'charity')),
    created_at TIMESTAMP DEFAULT NOW()
);

-- 12. MESSAGES Table (Simple messaging within workspaces/teams)
CREATE TABLE messages (
    id SERIAL PRIMARY KEY,
    team_id INTEGER NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    sender_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    content TEXT NOT NULL,
    sent_at TIMESTAMP DEFAULT NOW()
);

-- =================================================================
-- INDEXES FOR PERFORMANCE
-- =================================================================
CREATE INDEX idx_projects_charity_id ON projects(charity_id);
CREATE INDEX idx_teams_project_id ON teams(project_id);
CREATE INDEX idx_team_members_team_id ON team_members(team_id);
CREATE INDEX idx_activity_log_volunteer_id ON activity_log(volunteer_id);
-- Composite index is very important for polymorphic relations
CREATE INDEX idx_ratings_rated_entity ON ratings(rated_entity_id, rated_entity_type);
CREATE INDEX idx_messages_team_id ON messages(team_id);

