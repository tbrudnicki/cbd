# Introduction

This is a document serving purpose of input for design session for add SSO feature into currently existing application for SME banking.

# Business Context

We are building mobile application for small and medium businesses based in UAE. For first phase focused on delivering functional scope for single company operating on a single device. Now business is requesting delivering functionality which will allow to reduce friction of operating multiple companies on a single physical device. Plan is to add SSO feature in order to user authenticate once, and switch without effort between companies.

# Teminology

1. Company is identified by CompanyRim identifier
2. User is identified by PersonalRim identifier
3. Irim is identifier that groups CompanyRim and all associated PersonalRim identifiers

# Current state

Same customer (having single PersonalRim) when associated with some company has unqiue username associated with this company. After successfull authentication with dynamic challenges ping emits JWT token with this company-wise username. When logging to another company despite the user is the same (PersonalRim) the ping username is different. In order to switch company user needs to logout, pick other username and log into different company. All backend services are implemented in a way that the expect that JWT claims are related to single company.

# SSO implementation proposal

When user authenticate successfully to one company, he receives valid JWT token. When user wants to swith companies, mobile application could request in ping switching security context by passing valid JWT token, from there ping knows who it is, and he can check to which companies we would like to switch, and thanks to that we could avoid login again. Is this makes sense from security perspective. This will also change that when security context is changed in ping it will emit new JWT token for other company. Thank to this we won't have to rewrite new the entire backend sevices to accomodate no JWT token structure

