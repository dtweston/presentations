# iOS Anti-patterns

Learning by repeated failure

<br/>
Dave Weston
dweston@microsoft.com
http://github.com/dtweston

^ This talk started with an idea about exploring generic problems that often plague iOS projects. I've been doing iOS apps for a while, and JavaScript and .NET before that, so I have some experiences that I wanted to share. The trouble is that talking about things in the abstract is difficult. It's hard to care about a generic problem. So I'm here to tell you about what I'm doing, and what has and hasn't worked.

---

![inline 25%](http://upload.wikimedia.org/wikipedia/commons/thumb/d/db/Hello_my_name_is_sticker.svg/2000px-Hello_my_name_is_sticker.svg.png)

^ My name is Dave Weston. I'm an iOS developer, working for Yammer/Microsoft here in Redmond. Yammer is an enterprise social network, our mission is to make work better through communication, making companies more open and transparent. The iOS team builds a main app for iPhone/iPad and a separate chat app.

---
![100%](http://media.giphy.com/media/C1nEwzSrlVW80/giphy.gif)

^ We have developers in 3 different locations, and we ship to the App Store weekly. That means we have to move forward as quickly as possible while maintaining stability. New features, bug fixes, redesigns are always in progress. I don't claim that we have all the answers, but I'm here to talk about some of the things we've tried.

---

# Mobile apps are everywhere

^ You've seen the charts that show mobile usage growth as compared with the web. The app ecosystem has grown at a phenomenal rate.

---

# Apps are more complex

^ Local state, device features, everything has to be handled transparently for the user. The bar has been raised, and in order to impress, the apps we build have to get better.

---

# Apps stand alone

^ Apps today have to give users the full experience.  They can't be part of a larger offering, just ticking boxes on a chart. They have to be compelling, there are users out there who will only ever use your app

---

# Failures

(and successes)

^ Now we get to the fun stuff. What issues have we encountered at Yammer, and how have we tried to deal with them?

---

## Huge View Controllers

^ I'm willing to bet that anyone who is working on an iOS app is familiar with this problem. View controller classes seem to grow out of control.

---

### View controllers are glue

^ Apple doesn't give us good guidance here. The samples put all the logic in view controllers. View controllers are the delegate/datasource for everything. Sometimes that makes sense.

---

````objectivec
@implementation YMFeedViewController

#pragma mark - Header view
#pragma mark - fetching helper methods
#pragma mark - View lifecycle
#pragma mark - private methods
#pragma mark - Table view delegate
#pragma mark - Table view data source
#pragma mark - UIScrollViewDelegate
#pragma mark - YMFeedCellDelegate
#pragma mark - YMCardCellDelegate
#pragma mark - YMScrollableImageAttachmentViewDelegate
#pragma mark - YMThreadViewControllerDelegate
#pragma mark - iPad group transitions
#pragma mark - YMFetchedResultsTableControllerDelegate

@end
````
-
^ Here's a rough outline of the feed view controller from our app. It has a lot of responsibilities. What do we do about this? What have we done about this?

---

![fit left](signup.png)

### Move view code out of the view controller

SignupViewController => SignupViewController + SignupView

^ One good example of this is the signup view controller. We can move all of the view setup out of the controller, and into the view. The controller is just left to deal with the logic.

---

### Splitting responsibilities is good

````objectivec
@interface YKSignupView : UIScrollView

@property (nonatomic, strong) UILabel *firstNameLabel;
@property (nonatomic, strong) UILabel *lastNameLabel;
@property (nonatomic, strong) UILabel *emailLabel;
@property (nonatomic, strong) UILabel *passwordLabel;
@property (nonatomic, strong) YKErrorLabel *firstNameError;
@property (nonatomic, strong) YKErrorLabel *lastNameError;
@property (nonatomic, strong) YKErrorLabel *emailError;
@property (nonatomic, strong) YKErrorLabel *passwordError;
@property (nonatomic, strong) YKSimpleTextField *firstNameField;
@property (nonatomic, strong) YKSimpleTextField *lastNameField;
@property (nonatomic, strong) YKSimpleTextField *emailField;
@property (nonatomic, strong) YKSimpleTextField *passwordField;
@property (nonatomic, strong) YKSimplePrimaryButton *signupButton;
@property (nonatomic, strong) UILabel *termsOfUseLabel;
@property (nonatomic, strong) UIButton *termsOfUseButton;
@property (nonatomic, strong) YMKeyboardLayoutHelperView *keyboardHelper;

- (void)centerOnView:(UIView *)view;

@end

````
^ Both classes (SignupViewController and SignupView) are made simpler by separating them. The view just worries about view stuff, it doesn't know anything about the logic, and the same goes for the view controller. It has a simple interface to the view, and it is easy for the controller to tell the view what to display.

---

## ... do I detect a _however_?

^ This works well for those simple interfaces. It works less well when the subviews are more complicated, with more complicated behavior and logic interacting between different pieces.

---

![fit left](composer.png)

### And now the composer

- Address bar at the top
- Text view
- Tool bar at bottom

^ The composer is a wee bit more involved. The address bar can have a single group, and/or multiple individual people, with autocomplete. The text view has inline autocomplete and interacts with the text bar. The toolbar has a number of actions that can be formed that need to be bubbled up to the view controller's logic. You can mention groups in the address bar, but not inline, so that logic has to be handled slightly differently. When you mention someone in the message, they should also be added to the address bar by default, but you should be able to remove them from the address bar, and leave them mentioned in the message text.

---

````objectivec
- (BOOL)isMessageBodyEmpty
{
    return [self.composeTextView isEmpty];
}

- (NSString *)messageBodyText
{
    return [self.composeTextView originalText];
}

- (void)setMessageBodyText:(NSString *)text
{
    self.composeTextView.text = text;
}

- (NSAttributedString *)messageBodyAttributedText
{
    return self.composeTextView.attributedText;
}

- (void)setMessageBodyAttributedText:(NSAttributedString *)attributedText
{
    self.composeTextView.attributedText = attributedText;
}
````

^ You end up with code that is less obviously useful.

---

````objectivec
- (void)mentionableTextView:(YMMentionableTextView *)mentionableTextView didAddMentionedUser:(YDUser *)user
{
    [self.composeViewDelegate composeView:self didAddMentionedUser:user];
}

- (void)mentionableTextView:(YMMentionableTextView *)mentionableTextView didRemoveMentionedUser:(YDUser *)user
{
    [self.composeViewDelegate composeView:self didRemoveMentionedUser:user];
}

- (void)mentionableTextView:(YMMentionableTextView *)mentionableTextView didPasteImageData:(NSData *)data{
    [self.composeViewDelegate composeView:self didPasteImageData:data];
}
````

^ Or code that adds pain when debugging. Have you ever tried to fix a bug that involved following delegate calls backwards through three different levels of controllers? Not my idea of fun.

---

## Other solutions?

^ We have tried other solutions, with some success.

---

### separate data sources from view controllers

^ Pretty much everything on iOS is a table view controller. A nice way to get smaller classes is breaking the table view data source and delegate out to a separate class. Depending on how much interaction there is between the table view and other code in that view controller, this can be a big win. But it does tend to tie you pretty tightly to the UITableView paradigm, making it harder to switch to something else.

---

### parent/child view controller relationships

^ Parent/child view controllers are complex! The framework has some interesting quirks (shall we say) about how it handles this case. Also, view controllers are pretty heavyweight things! There are times where this is the right solution, but we can't always rely on this.

---

### views that understand the model layer

^ This solution moves code out of the view controller and into the view. You can just tell the view, here, make yourself represent this thing. Which works fine, but you lose some of the benefits of the MVC setup. You also make that view less reusable. Sometimes that's not something you should worry about because the view just won't be reused, but it has caused problems for us. View-based testing can also be a lot more complicated when using this method. If you have to setup a complicated model relationship to test something, it's likely that you just won't do it.

---

### WE'RE _doomed_ 
### we're all _doooooomed_

^ So do we just give up? How do we handle this? YES, we should all give up. Programming is hard, and it's not getting any easier.

---

### `NSObject` controllers

^ The C in MVC that inherits from NSObject instead of UIViewController. Lighter-weight, easy to test. Easier to compose with other controllers. What's the verdict? Too early to tell.

---

## Tests

^ I mentioned testing briefly earlier. iOS testing at Yammer has had a bit of a rocky road. We are getting better though.

---

## plan 1

### KIF (Keep It Functional)

^ KIF is a framework originally built by Square, that is on github. We built some incredibly fragile and painful tests in KIF that in retrospect probably only slowed us down

---

## plan 2

### Kiwi (BDD testing)

```objectivec
describe(@"Math", ^{
  context(@"Addition", ^{
    it(@"is commutative", ^{
      [[theValue(3+2) should] equal:theValue(2+3)];
    });
  });
});
```

^ For those of you familiar with Ruby, Kiwi is like an rspec clone for Objective C. BDD stands for behavior driven development. Kiwi supports mocking well and it has syntax that can read like simple English text. We have 1200 tests of our code. Lots of tests around things that are easy to test, fewer tests around things like view controllers/views.

---

## More/better testing

### How to improve test coverage?

^ As I said we have lots of tests around things that are easy to test. But how do we get better?

---

### Two options

- bigger tests, more setup
- smaller units to be tested

^ This is an obvious choice. We are splitting code out into smaller classes that do one thing. For example...

---

![fit left](message_formatting.png)

### message formatting

^ Message formatting used to be handled by the message model object itself. The message would query parts of itself to generate an attributed string. Now we have a pipeline of different stages that are tested and can be modified independently.

---

### mention detection

```objectivec
describe(@"YMMentionDetector", ^{

    it(@"detects mentions due to capitalization", ^{
      ...
    });

    it(@"detects mentions due to @ symbol", ^{
      ...
    });

    it(@"detects mentions when the cursor is moved back", ^{
      ...
    });

    it(@"detects symbol mentions when they are at the beginning of the text.", ^{
      ...
    });
});

```

^ This code was previously tangled up in the compose view controller. By putting it into a separate class, it's easy to test its behavior and modify it as needed safely. There's even a "fishy behavior" section which captures corner cases that were implemented by the original code, but should be reviewed to make sure they actually make sense. Now the conclusion, this is where I wrap everything up and try to end on a happy philosophical note. Bear with me.

---

![fit](churn.png)

### Not all code changes happen
### at the same rate

^ This is a histogram of changes in our main iOS app per file. As you can see, some files have a disproportionate amount of churn. This graph is typical of any source code repo, and lots of other processes. Code churn indicates areas that are in flux, because of changing requirements, new features, market changes, whatever the case may be. 

---

## SCIENCE

^ Studies have shown that areas of high churn are more likely to see bugs.

---

### Experiments

![left fit](experiments.png)

^ One way we ensure fewer regressions is by rolling out code behind an experiment. This keeps everyone honest, because all developers stay close to master. Bugs don't have a chance to hide for long. Integrations are always easy.

---

# [fit] QED

# [fit] smaller classes => more classes => less bugs => happier users

^ Smaller classes let us improve our tests. More classes let us encapsulate more behavior behind an experiment. Leading to less regressions. Better tests, less regressions, fewer bugs, happier users.. etc.

---

# [fit] WORLD PEACE



