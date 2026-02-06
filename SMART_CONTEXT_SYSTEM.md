# Smart Context System for Group Chat Threads

## Overview

The Smart Context System automatically analyzes conversation threads to provide intelligent context based on:

1. **Conversation Type**: Single user vs group chat detection
2. **Theme & SubTheme**: Context from `assistanceAndGreetings.py` mappings
3. **Group Chat Instructions**: Ensures AI uses names instead of pronouns
4. **Dynamic Integration**: Automatically integrates into all response generators

## Key Features

### 🔍 Automatic Detection
- **Group Chat Detection**: Analyzes `userDetails.otherUsers` to determine if it's a group conversation
- **Theme Context**: Maps themes and subthemes to relevant conversation contexts
- **User Analysis**: Extracts user names and relationships for proper addressing

### 🎯 Context-Aware Responses
- **Theme-Specific Focus**: Adds relevant context based on conversation theme (e.g., budget planning, event coordination)
- **Group Benefits**: Highlights collaborative aspects for group chats
- **Smart Instructions**: Provides different guidance for group vs individual conversations
- **Conversation Context**: Natural, conversational context that complements existing role definitions

### 💬 Conversation Context Feature
- **Single User**: "Conversation Context: This is a [subTheme] conversation focused on [focus]..."
- **Group Chat**: "Conversation Context: This is a [subTheme] group chat with [names] focused on [focus]..."
- **Non-Conflicting**: Complements existing role definitions rather than overriding them
- **Dynamic Content**: Automatically includes participant names and theme-specific purposes

#### Conversation Context Examples:

**findDeals Group Chat:**
```
Conversation Context: This is a findDeals group chat with Sarah, Mike and Lisa focused on finding deals, discount codes, and money-saving opportunities. Prioritize sharing deals and discounts, group buying for better prices and optimize responses for group collaboration and shared decision-making.
```

**destroyDebt Single User:**
```
Conversation Context: This is a destroyDebt conversation focused on debt payoff planning, repayment tracking, and debt management. Prioritize debt elimination and repayment strategies and provide personalized guidance.
```

**planAnEvent Group Chat:**
```
Conversation Context: This is a planAnEvent group chat with Emma and David focused on event budgeting, cost planning, and expense tracking for events. Prioritize coordinating group events, shared event planning and costs and optimize responses for group collaboration and shared decision-making.
```

#### Non-Conflicting Integration Example:

**Original Prompt:**
```
You are Barry, a helpful AI assistant. Be friendly and supportive.
```

**Enhanced with Smart Context:**
```
You are Barry, a helpful AI assistant. Be friendly and supportive.

Conversation Context: This is a findDeals group chat with Alice and Bob focused on finding deals, discount codes, and money-saving opportunities. Prioritize sharing deals and discounts, group buying for better prices and optimize responses for group collaboration and shared decision-making.
```

**Result:** The original role definition ("You are Barry...") remains intact, while the smart context adds specific conversation guidance without conflict.

### 👥 Group Chat Optimization
- **Smart Name Usage**: Instructs AI on proper addressing in group contexts
- **Clear Attribution**: Ensures recommendations specify who they're for
- **Participant Clarity**: Maintains clear communication in multi-user scenarios

#### Group Chat Communication Rules
- ✅ **Direct addressing**: "Hey Sarah, your budget looks good" (clear who "your" refers to)
- ❌ **Ambiguous reference**: "Your budget looks good" (whose budget?)
- ✅ **Possessive names**: "Sarah's budget" when not directly addressing
- ✅ **Specific recommendations**: "This deal is perfect for Mike"
- ❌ **Vague recommendations**: "This deal is perfect for you" (who?)

## Integration Points

The smart context system is automatically integrated into:

- ✅ **`base_gpt.py`** - Analysis prompts
- ✅ **`direct_response.py`** - Direct response prompts
- ✅ **`static_response_generator.py`** - Static response prompts
- ✅ **`datetime_response_generator.py`** - Datetime response prompts

## Theme Mappings

The system supports all themes and subthemes from `assistanceAndGreetings.py`:

### 💰 manageMoney
- **moveMoney**: Financial transfers and payments
- **poolMoney**: Group savings and contribution management
- **raiseMoney**: Fundraising and financial goal achievement
- **monitorMoney**: Financial tracking and spending analysis
- **growMoney**: Savings growth and financial planning
- **shareMoney**: Shared expenses and cost management
- **destroyDebt**: Debt elimination and repayment strategies

### 💼 earningAndSaving
- **earnMoney**: Income generation and employment opportunities
- **findDeals**: Discount hunting and savings opportunities
- **navigateGigLife**: Gig economy and freelance work management
- **exploreCrypto**: Cryptocurrency learning and portfolio management

### 🎉 lifestyleAndEvent
- **planAnEvent**: Event planning and budget management
- **styleOnABudget**: Budget-conscious fashion and lifestyle choices
- **huntExperiences**: Affordable experiences and activity planning

### 🤝 socialAndCommunity
- **createCashCommunity**: Financial community building and support

## Usage Examples

### Single User Thread
```python
request_info = {
    "userDetails": {
        "primaryUser": {
            "userId": "user123",
            "name": "John Smith",
            "location": "New York"
        },
        "otherUsers": []  # Empty = single user
    },
    "theme": "manageMoney",
    "subTheme": "budgetPlanning"
}

context = SmartContextGenerator.generate_smart_context(request_info)
# Result: Theme context only, no group instructions
```

### Group Chat Thread
```python
request_info = {
    "userDetails": {
        "primaryUser": {
            "userId": "user123",
            "name": "Sarah Johnson",
            "location": "San Francisco"
        },
        "otherUsers": [
            {
                "userId": "user456",
                "name": "Mike Chen",
                "location": "Los Angeles"
            },
            {
                "userId": "user789",
                "name": "Lisa Rodriguez",
                "location": "Chicago"
            }
        ]
    },
    "theme": "lifestyleAndEvent",
    "subTheme": "planAnEvent"
}

context = SmartContextGenerator.generate_smart_context(request_info)
# Result: Theme context + group chat instructions
```

## Generated Context Examples

### Single User Context
```
Conversation Context: This is a poolMoney conversation focused on collective savings, group contributions, and shared financial goals. Prioritize group savings and contribution management and provide personalized guidance.

CONVERSATION THEME CONTEXT:
- Primary Focus: collective savings, group contributions, and shared financial goals
- Context Area: group savings and contribution management
```

### Group Chat Context
```
Conversation Context: This is a planAnEvent group chat with Sarah Johnson, Mike Chen and Lisa Rodriguez focused on event budgeting, cost planning, and expense tracking for events. Prioritize coordinating group events, shared event planning and costs and optimize responses for group collaboration and shared decision-making.

CONVERSATION THEME CONTEXT:
- Primary Focus: event budgeting, cost planning, and expense tracking for events
- Context Area: event planning and budget management
- Group Benefits: coordinating group events, shared event planning and costs
- Optimize responses for group collaboration and shared decision-making

CRITICAL GROUP CHAT INSTRUCTIONS:
- This is a group conversation with 3 participants: Sarah Johnson, Mike Chen and Lisa Rodriguez
- ALWAYS be clear about who you're referring to or addressing
- When directly addressing someone, use their name first, then "your" is okay: "Hey Sarah, your budget" ✅
- AVOID ambiguous "your" without clear attribution: "your budget" (whose?) ❌ 
- When not directly addressing them, use possessive names: "Sarah's budget", "Mike's preference"
- NEVER use unclear pronouns like "he", "she", "they", "them" without names
- When making recommendations, be clear about who it's for: "This deal would be perfect for Mike" not "This would be perfect for you"
- Use names to clarify context: "Based on what Sarah mentioned..." not "Based on what you mentioned..."
- For group activities, specify participants: "This could work for both Alex and Jordan" not "This could work for both of you"
```

## API Reference

### SmartContextGenerator

#### `generate_smart_context(request_info: Dict[str, Any]) -> Dict[str, Any]`
Generates comprehensive smart context for the conversation.

**Parameters:**
- `request_info`: Request information containing user details, theme, subtheme

**Returns:**
```python
{
    "conversation_analysis": {
        "is_group_chat": bool,
        "total_users": int,
        "user_names": List[str],
        "primary_user_name": str,
        "other_user_count": int
    },
    "theme_context": Dict[str, str] | None,
    "group_instructions": str,
    "theme_context_text": str,
    "role_description": str,  # NEW: Natural language conversation context (non-conflicting)
    "full_context": str,
    "has_context": bool,
    "is_group_chat": bool,
    "user_names": List[str],
    "theme": str,
    "sub_theme": str
}
```

#### `inject_context_into_prompt(base_prompt: str, smart_context: Dict[str, Any]) -> str`
Injects smart context into an existing prompt.

#### `get_context_for_request(request_info: Dict[str, Any]) -> str`
Simple method to get context string ready for prompt injection.

## Benefits

### 🎯 For Single User Threads
- **Focused Context**: Provides relevant theme-specific guidance
- **Personal Approach**: Maintains direct, personal communication style
- **Specialized Knowledge**: Leverages theme expertise for better responses

### 👥 For Group Chat Threads
- **Clear Communication**: Eliminates pronoun confusion in multi-user scenarios
- **Collaborative Focus**: Emphasizes group benefits and coordination
- **Participant Clarity**: Ensures everyone knows who responses are directed to
- **Enhanced Coordination**: Facilitates better group decision-making

### 🚀 System-Wide Impact
- **Automatic Integration**: Works across all response generators without manual intervention
- **Non-Conflicting**: Complements existing role definitions without overriding them
- **Consistent Behavior**: Ensures uniform smart context application
- **Improved User Experience**: Provides more relevant, contextual responses
- **Scalable Design**: Easy to add new themes and conversation types

## Testing

Run the demonstration to see the system in action:

```bash
python examples/smart_context_demo.py
```

This will show:
- Single user vs group chat detection
- Different theme contexts
- **NEW: Natural language conversation context** for different scenarios
- Context injection examples showing non-conflicting integration
- Group vs single user instruction differences

## Future Enhancements

Potential areas for expansion:

1. **Dynamic Theme Detection**: Auto-detect themes from conversation content
2. **Context Memory**: Remember conversation context across sessions
3. **User Preference Learning**: Adapt context based on user interaction patterns
4. **Multi-Language Support**: Extend context generation to multiple languages
5. **Custom Context Rules**: Allow for custom context rules per user or group

## Implementation Notes

- The system is **zero-impact** on existing functionality - it only adds context when beneficial
- **Non-conflicting design** - uses "Conversation Context:" prefixes to complement, not override, existing role definitions
- **Performance optimized** - context generation is fast and doesn't add significant overhead
- **Backward compatible** - works with existing request structures without requiring changes
- **Error resilient** - gracefully handles missing or invalid data
- **Extensible design** - easy to add new themes, conversation types, or context rules 